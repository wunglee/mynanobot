# nanobot 项目架构全景图

> 本文档从宏观到微观全面解析 nanobot 项目架构，帮助开发者快速理解项目设计思想和实现细节。

## 📋 目录

1. [项目概述](#项目概述)
2. [架构总览](#架构总览)
3. [核心子系统](#核心子系统)
4. [技术栈](#技术栈)
5. [模块详解](#模块详解)
6. [数据流](#数据流)
7. [扩展机制](#扩展机制)
8. [部署架构](#部署架构)

---

## 项目概述

**nanobot** 是一个超轻量级个人 AI 助手，灵感来源于 Clawdbot/OpenClaw。它在约 **4,000 行代码**中实现了核心 Agent 功能，比 OpenClaw 的 430k+ 行代码精简 99%。

### 核心特性

| 特性 | 描述 |
|------|------|
| **超轻量级** | ~4,000 行核心代码，易于理解和修改 |
| **多提供商支持** | OpenRouter、OpenAI、Anthropic、DeepSeek、Qwen 等 |
| **多渠道** | Telegram、Discord、WhatsApp、Feishu |
| **本地优先** | 支持 vLLM 本地模型部署 |
| **任务调度** | 自然语言定时任务（Cron）|
| **语音支持** | Groq Whisper 语音转文字 |

---

## 架构总览

### 系统架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              客户端层 (Clients)                               │
├─────────────┬─────────────┬─────────────┬─────────────────────────────────────┤
│  Telegram   │  Discord    │  WhatsApp   │  Feishu (飞书)                       │
│  (Bot API)  │  (Bot API)  │  (Baileys)  │  (WebSocket)                         │
└──────┬──────┴──────┬──────┴──────┬──────┴──────────┬──────────────────────────┘
       │             │             │                 │
       └─────────────┴─────────────┴─────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Gateway 网关服务                                 │
│                      (WebSocket Server + HTTP API)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Channel │  │  Agent   │  │  Session │  │   Tool   │  │     Cron     │  │
│  │  Manager │  │  Loop    │  │  Manager │  │  System  │  │   Service    │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Event   │  │  Config  │  │  Memory  │  │  SubAgent│  │   Provider   │  │
│  │   Bus    │  │  Manager │  │  System  │  │  Runner  │  │   (LiteLLM)  │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              LLM Providers                                  │
├──────────────┬──────────────┬──────────────┬──────────────┬─────────────────┤
│  OpenRouter  │   OpenAI     │  Anthropic   │  DeepSeek    │     vLLM        │
│  (Gateway)   │   (GPT)      │  (Claude)    │              │  (Local)        │
└──────────────┴──────────────┴──────────────┴──────────────┴─────────────────┘
```

### 分层架构

```
┌─────────────────────────────────────────────────────────┐
│  Layer 4: 应用层 (Interfaces)                            │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │  CLI    │ │ Telegram│ │ Discord │ │  Web    │       │
│  │         │ │   Bot   │ │   Bot   │ │   UI    │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
├─────────────────────────────────────────────────────────┤
│  Layer 3: 服务层 (Services)                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │  Agent  │ │ Channel │ │  Tool   │ │  Cron   │       │
│  │  Loop   │ │ Manager │ │ System  │ │ Service │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
├─────────────────────────────────────────────────────────┤
│  Layer 2: 核心层 (Core)                                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │  Event  │ │ Session │ │  Config │ │  Skills │       │
│  │   Bus   │ │ Manager │ │ Manager │ │  Loader │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
├─────────────────────────────────────────────────────────┤
│  Layer 1: 基础设施层 (Infrastructure)                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │  Utils  │ │ Provider│ │  Memory │ │  Bridge │       │
│  │         │ │(LiteLLM)│ │  Store  │ │(WhatsApp│       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
└─────────────────────────────────────────────────────────┘
```

---

## 核心子系统

### 1. Gateway 网关服务

Gateway 是 nanobot 的核心，提供 HTTP API 和 WebSocket 服务。

```python
# Gateway 核心职责
class Gateway:
    # HTTP 服务器
    http_server: HTTPServer
    
    # WebSocket 服务器
    ws_server: WebSocketServer
    
    # 会话管理
    sessions: SessionManager
    
    # 渠道适配器注册表
    channels: ChannelRegistry
    
    # 工具注册表
    tools: ToolRegistry
    
    # 定时任务服务
    cron: CronService
    
    # 配置管理
    config: ConfigManager
    
    # 事件总线
    event_bus: EventBus
```

**关键文件**:
- `nanobot/channels/manager.py` - 渠道管理器
- `nanobot/bus/events.py` - 事件总线
- `nanobot/bus/queue.py` - 消息队列

### 2. 渠道系统 (Channels)

统一的多渠道消息处理架构。

```
┌─────────────────────────────────────────────────────┐
│                  Channel Manager                    │
├─────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────┐ │
│  │Telegram │  │ Discord │  │WhatsApp │  │ Feishu │ │
│  │ Handler │  │ Handler │  │ Handler │  │ Handler│ │
│  └────┬────┘  └────┬────┘  └────┬────┘  └───┬────┘ │
│       │            │            │           │      │
│  ┌────┴────────────┴────────────┴───────────┴────┐ │
│  │              Channel Base Class               │ │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐         │ │
│  │  │  send   │ │receive  │ │ format  │         │ │
│  │  └─────────┘ └─────────┘ └─────────┘         │ │
│  └────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

**核心渠道**:
| 渠道 | 库/技术 | 适配器位置 |
|------|---------|-----------|
| Telegram | python-telegram-bot | `nanobot/channels/telegram.py` |
| Discord | discord.py | `nanobot/channels/discord.py` |
| WhatsApp | Node.js Bridge | `nanobot/channels/whatsapp.py` |
| Feishu | lark-oapi | `nanobot/channels/feishu.py` |

**关键文件**:
- `nanobot/channels/base.py` - 渠道基类
- `nanobot/channels/manager.py` - 渠道管理器

### 3. Agent 运行时

基于 LiteLLM 的智能体运行时系统。

```
┌─────────────────────────────────────────────────────────────┐
│                     Agent Loop                              │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Context   │  │    Tool     │  │   Memory    │         │
│  │   Builder   │  │  Executor   │  │   Store     │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Provider   │  │   Model     │  │   Skills    │         │
│  │  (LiteLLM)  │  │  Selection  │  │   Loader    │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │    Bash     │  │    Web      │  │    File     │         │
│  │    Tool     │  │   Search    │  │   Tools     │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

**关键文件**:
- `nanobot/agent/loop.py` - Agent 主循环
- `nanobot/agent/context.py` - 上下文构建器
- `nanobot/agent/tools/` - 工具实现

### 4. 工具系统 (Tools)

```python
# 工具系统架构
class ToolSystem:
    # 内置工具
    built_in: {
        bash: BashTool,           # 命令执行
        read: FileReadTool,       # 文件读取
        write: FileWriteTool,     # 文件写入
        search: WebSearchTool,    # 网页搜索
        spawn: SubAgentTool,      # 子代理
    }
    
    # 技能工具
    skills: SkillTool[]
```

### 5. 定时任务系统 (Cron)

基于 croniter 的定时任务服务。

```
┌─────────────────────────────────────────────────────────────┐
│                     Cron Service                            │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Job Scheduler                          │   │
│  │  • Cron Expression Parser (croniter)               │   │
│  │  • Natural Language Scheduling                     │   │
│  │  • Next Run Calculation                            │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Add Job   │  │  List Jobs  │  │ Remove Job  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

**关键文件**:
- `nanobot/cron/service.py` - 定时任务服务
- `nanobot/cron/types.py` - 类型定义

---

## 技术栈

### 后端技术栈

| 类别 | 技术 | 用途 |
|------|------|------|
| **运行时** | Python 3.11+ | 主运行时 |
| **类型系统** | Type Hints + Pydantic | 类型安全 |
| **包管理** | pip / uv | 包管理器 |
| **构建** | hatchling | 构建系统 |
| **Lint** | ruff | 代码检查 |
| **测试** | pytest | 测试框架 |

### 核心依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| `typer` | >=0.9.0 | CLI 框架 |
| `litellm` | >=1.0.0 | 统一 LLM 接口 |
| `pydantic` | >=2.0.0 | 数据验证 |
| `websockets` | >=12.0 | WebSocket 服务 |
| `httpx` | >=0.25.0 | HTTP 客户端 |
| `loguru` | >=0.7.0 | 日志系统 |
| `rich` | >=13.0.0 | 终端美化 |
| `croniter` | >=2.0.0 | Cron 表达式 |
| `python-telegram-bot` | >=21.0 | Telegram Bot |

### 渠道技术栈

| 平台 | 技术 | 用途 |
|------|------|------|
| **Telegram** | python-telegram-bot | Bot API |
| **Discord** | discord.py | Bot API |
| **WhatsApp** | Node.js Bridge | Baileys 桥接 |
| **Feishu** | lark-oapi | 飞书 API |

---

## 模块详解

### 目录结构

```
nanobot/
├── __init__.py              # 包入口
├── __main__.py              # 可执行入口
├── 
├── agent/                   # 🧠 Agent 核心逻辑
│   ├── __init__.py
│   ├── loop.py              # Agent 主循环
│   ├── context.py           # 提示词构建器
│   ├── memory.py            # 持久化记忆
│   ├── skills.py            # 技能加载器
│   ├── subagent.py          # 后台任务执行
│   └── tools/               # 内置工具
│       ├── __init__.py
│       ├── bash.py          # Bash 工具
│       ├── file.py          # 文件操作
│       └── web.py           # 网页搜索
│
├── skills/                  # 🎯 内置技能
│   ├── README.md
│   └── ...                  # GitHub、天气、tmux 等
│
├── channels/                # 📱 渠道集成
│   ├── __init__.py
│   ├── base.py              # 渠道基类
│   ├── manager.py           # 渠道管理器
│   ├── telegram.py          # Telegram
│   ├── discord.py           # Discord
│   ├── whatsapp.py          # WhatsApp
│   └── feishu.py            # 飞书
│
├── bus/                     # 🚌 消息总线
│   ├── __init__.py
│   ├── events.py            # 事件定义
│   └── queue.py             # 消息队列
│
├── cron/                    # ⏰ 定时任务
│   ├── __init__.py
│   ├── service.py           # 调度服务
│   └── types.py             # 类型定义
│
├── heartbeat/               # 💓 主动唤醒
│   ├── __init__.py
│   └── service.py           # 心跳服务
│
├── providers/               # 🤖 LLM 提供商
│   ├── __init__.py
│   ├── base.py              # 提供商基类
│   ├── litellm_provider.py  # LiteLLM 集成
│   └── transcription.py     # 语音转录
│
├── session/                 # 💬 会话管理
│   ├── __init__.py
│   └── manager.py           # 会话管理器
│
├── config/                  # ⚙️ 配置系统
│   ├── __init__.py
│   ├── loader.py            # 配置加载
│   └── schema.py            # Pydantic Schema
│
├── cli/                     # 🖥️ 命令行
│   ├── __init__.py
│   └── commands.py          # CLI 命令
│
└── utils/                   # 🛠️ 工具函数
    ├── __init__.py
    └── helpers.py           # 辅助函数
```

---

## 数据流

### 1. 入站消息流

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Channel   │────▶│   Channel   │────▶│   Event     │────▶│   Session   │
│  (Telegram  │     │   Handler   │     │    Bus      │     │   Manager   │
│   etc.)     │     │             │     │             │     │             │
└─────────────┘     └─────────────┘     └─────────────┘     └──────┬──────┘
                                                                   │
                                                                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Channel   │◀────│   Response  │◀────│    Agent    │◀────│    Agent    │
│   (Reply)   │     │   Handler   │     │    Loop     │     │   Runtime   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

### 2. 工具调用流

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    Agent    │────▶│   Tool      │────▶│   Tool      │────▶│   Execute   │
│   Runtime   │     │   Request   │     │   Router    │     │   Tool      │
└─────────────┘     └─────────────┘     └─────────────┘     └──────┬──────┘
                                                                   │
┌─────────────┐     ┌─────────────┐     ┌─────────────┐           │
│    Agent    │◀────│   Format    │◀────│   Collect   │◀──────────┘
│   Runtime   │     │   Result    │     │   Result    │
└─────────────┘     └─────────────┘     └─────────────┘
```

### 3. Event Bus 流程

```
┌─────────────┐                              ┌─────────────┐
│   Channel   │◀────────── Event ───────────▶│   Agent     │
│   Handler   │         nanobot.bus          │    Loop     │
└─────────────┘                              └──────┬──────┘
       │                                            │
       │    ┌──────────┐  ┌──────────┐  ┌────────┐ │
       ├───▶│ message  │  │  tool    │  │  cron  │ │
       │    │ received │  │  invoke  │  │  tick  │ │
       │    └──────────┘  └──────────┘  └────────┘ │
       │                                            │
       │    ┌──────────┐  ┌──────────┐            │
       └───▶│ session  │  │  error   │            │
            │  start   │  │  event   │            │
            └──────────┘  └──────────┘            │
                                                  │
```

---

## 扩展机制

### 1. 渠道扩展

创建新渠道的步骤：

```python
# 1. 继承渠道基类
from nanobot.channels.base import Channel

class MyChannel(Channel):
    def __init__(self, config: dict):
        super().__init__(config)
        self.client = None
    
    # 2. 实现启动方法
    async def start(self):
        """启动渠道连接"""
        pass
    
    # 3. 实现停止方法
    async def stop(self):
        """停止渠道连接"""
        pass
    
    # 4. 实现发送消息
    async def send_message(self, chat_id: str, text: str, **kwargs):
        """发送消息"""
        pass
    
    # 5. 处理接收消息
    async def on_message(self, message: dict):
        """处理收到的消息"""
        await self.emit("message", message)
```

### 2. 工具扩展

```python
# 通过装饰器注册工具
from nanobot.agent.tools import tool

@tool(name="my_tool", description="My custom tool")
async def my_tool(input: str) -> str:
    """工具实现"""
    return f"Processed: {input}"
```

### 3. 技能扩展

```python
# 创建技能目录结构
skills/
└── my_skill/
    ├── __init__.py
    ├── README.md
    └── tools.py
```

---

## 部署架构

### 本地部署

```
┌─────────────────────────────────────────────────────────────┐
│                     用户设备 (Linux/Mac/Win)                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  nanobot (Python)                      │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │  │
│  │  │Telegram │ │ Discord │ │WhatsApp │ │  Cron   │ ... │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘     │  │
│  └───────────────────────────────────────────────────────┘  │
│                           │                                  │
│  ┌────────────────────────┴────────────────────────────┐    │
│  │              ~/.nanobot/ 数据目录                    │    │
│  │  ├─ config.json          # 配置文件                 │    │
│  │  ├─ workspace/           # 工作区                   │    │
│  │  └─ memory/              # 记忆数据                 │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Docker 部署

```yaml
# Docker 部署
docker run -v ~/.nanobot:/root/.nanobot -p 18790:18790 nanobot gateway
```

---

## 配置体系

### 配置层次

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: 默认配置 (代码内置)                                │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: 全局配置文件 ~/.nanobot/config.json               │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: 环境变量 NANOBOT_*                                 │
├─────────────────────────────────────────────────────────────┤
│  Layer 4: CLI 参数 --flag                                    │
└─────────────────────────────────────────────────────────────┘
```

### 核心配置项

```python
class NanobotConfig(BaseModel):
    # 提供商配置
    providers: dict[str, ProviderConfig]
    
    # Agent 配置
    agents: AgentConfig
    
    # 渠道配置
    channels: ChannelsConfig
    
    # 工具配置
    tools: ToolsConfig
    
    # 安全配置
    restrict_to_workspace: bool = False
```

---

## 安全模型

### 默认安全策略

```
┌─────────────────────────────────────────────────────────────┐
│                     安全层级                                 │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: 工作区隔离                                         │
│  • restrictToWorkspace 限制工具访问范围                      │
│  • 防止路径遍历攻击                                          │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: 允许列表 (allowFrom)                              │
│  • 基于用户 ID 的白名单                                      │
│  • 空列表 = 允许所有人                                       │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: 工具权限                                           │
│  • 文件操作限制在工作区内                                    │
│  • Shell 命令可配置禁用                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 开发指南

### 本地开发流程

```bash
# 1. 克隆仓库
git clone https://github.com/HKUDS/nanobot.git
cd nanobot

# 2. 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 3. 安装依赖
pip install -e ".[dev]"

# 4. 初始化配置
nanobot onboard

# 5. 编辑配置添加 API 密钥
vim ~/.nanobot/config.json

# 6. 运行测试
pytest

# 7. 启动网关
nanobot gateway
```

### 调试技巧

```bash
# Agent 调试
nanobot agent -m "test message"

# 开启详细日志
export LOGURU_LEVEL=DEBUG
nanobot gateway

# 检查状态
nanobot status
```

---

## 总结

nanobot 采用**轻量级架构**和**模块化设计**，具有以下核心特点：

1. **精简高效** - 约 4,000 行代码实现核心功能
2. **统一抽象** - LiteLLM 统一多提供商接口
3. **事件驱动** - 基于事件总线的松耦合架构
4. **易于扩展** - 清晰的插件化设计
5. **本地优先** - 数据保留在用户设备上

### 架构演进方向

- 更多渠道适配器
- 长期记忆系统
- 多模态支持
- 自我改进机制

---

*文档生成时间: 2025年*
*版本: nanobot 0.1.3*
