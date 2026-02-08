# nanobot 知识图谱大纲

> 本文件列出 nanobot 项目中涉及的所有技术知识点，作为学习路径参考。

---

## 一、Python 基础与现代特性

### 1.1 语言核心
- [ ] Python 3.11+ 新特性
- [ ] 类型注解（Type Hints）
- [ ] TypeVar 与泛型
- [ ] Protocol 与结构子类型
- [ ] dataclasses 数据类
- [ ] 异步编程（async/await）
- [ ] 事件循环机制
- [ ] 上下文管理器（with/async with）

### 1.2 Python 标准库
- [ ] `pathlib` 路径处理
- [ ] `asyncio` 异步库
- [ ] `json` JSON 处理
- [ ] `logging` 日志系统
- [ ] `subprocess` 进程管理
- [ ] `tempfile` 临时文件
- [ ] `sqlite3` 数据库
- [ ] `re` 正则表达式

### 1.3 现代 Python 特性
- [ ] 模式匹配（match/case）
- [ ] 联合类型（Union | X）
- [ ] 可选类型（Optional | None）
- [ ] walrus 运算符（:=）
- [ ] f-string 高级用法
- [ ] 字典合并（| 运算符）

---

## 二、CLI 开发

### 2.1 Typer
- [ ] App 实例创建
- [ ] 命令定义（Commands）
- [ ] 选项定义（Options）
- [ ] 参数解析（Arguments）
- [ ] 帮助信息生成
- [ ] 回调函数

### 2.2 交互式 CLI
- [ ] Rich（终端美化）
- [ ] Prompt 交互提示
- [ ] 进度条和加载动画
- [ ] 表格输出格式化
- [ ] 日志美化（loguru）

### 2.3 CLI 架构模式
- [ ] 命令分组
- [ ] 依赖注入
- [ ] 配置管理集成

---

## 三、Web 开发

### 3.1 Web 框架
- [ ] FastAPI（现代 Web 框架）
- [ ] ASGI 应用
- [ ] 中间件概念
- [ ] 路由系统
- [ ] 请求/响应处理
- [ ] 依赖注入系统

### 3.2 WebSocket
- [ ] websockets 库使用
- [ ] WebSocket 协议基础
- [ ] 实时双向通信
- [ ] 连接管理和心跳

### 3.3 HTTP 客户端
- [ ] httpx（异步 HTTP 客户端）
- [ ] aiohttp
- [ ] 请求拦截和重试
- [ ] 流式响应处理

---

## 四、即时通讯平台集成

### 4.1 平台 SDK
- [ ] **Telegram**: python-telegram-bot
- [ ] **Discord**: discord.py
- [ ] **WhatsApp**: bridge 模式（Node.js Baileys）
- [ ] **Feishu**: lark-oapi SDK

### 4.2 消息处理
- [ ] Webhook 处理
- [ ] 长轮询（Long Polling）
- [ ] 消息格式解析
- [ ] 富媒体消息（图片、文件、语音）
- [ ] 消息路由系统

---

## 五、AI/LLM 集成

### 5.1 AI 提供商 SDK
- [ ] **LiteLLM**: 统一 LLM 接口
- [ ] **OpenAI**: GPT API
- [ ] **Anthropic**: Claude API
- [ ] **OpenRouter**: 多模型网关
- [ ] **vLLM**: 本地模型部署
- [ ] **DashScope**: Qwen API

### 5.2 Agent 框架
- [ ] Agent Loop 设计
- [ ] 工具调用（Tool Calling）
- [ ] Function Calling 模式
- [ ] 多轮对话管理
- [ ] 上下文窗口管理

### 5.3 提示工程
- [ ] 系统提示词设计
- [ ] 少样本提示（Few-shot）
- [ ] 思维链（Chain-of-Thought）
- [ ] 结构化输出（JSON Mode）

---

## 六、数据验证与类型安全

### 6.1 Schema 验证
- [ ] **Pydantic v2**: 运行时类型验证
- [ ] **Pydantic Settings**: 配置管理
- [ ] BaseModel 与 Field
- [ ] ConfigDict 配置

### 6.2 类型安全
- [ ] mypy 类型检查
- [ ] 泛型模型
- [ ] 自定义验证器
- [ ] 序列化/反序列化

---

## 七、数据存储

### 7.1 JSON 存储
- [ ] JSON 文件读写
- [ ] 原子写入模式

### 7.2 SQLite
- [ ] sqlite3 基础
- [ ] 嵌入式数据库
- [ ] 简单查询与索引

### 7.3 内存管理
- [ ] 会话状态管理
- [ ] 持久化记忆系统

---

## 八、浏览器自动化

### 8.1 Playwright
- [ ] 浏览器控制
- [ ] 页面导航和交互
- [ ] 截图和 PDF 生成
- [ ] 异步 API 使用

### 8.2 网页解析
- [ ] readability-lxml
- [ ] 内容提取
- [ ] HTML 解析

---

## 九、任务调度

### 9.1 定时任务
- [ ] croniter（Cron 表达式）
- [ ] APScheduler
- [ ] 自然语言任务调度

### 9.2 后台任务
- [ ] asyncio.create_task
- [ ] 后台执行器
- [ ] 任务队列

---

## 十、架构设计模式

### 10.1 设计模式
- [ ] 依赖注入（DI）
- [ ] 工厂模式
- [ ] 观察者模式（Pub/Sub）
- [ ] 策略模式
- [ ] 适配器模式

### 10.2 架构风格
- [ ] 事件驱动架构
- [ ] 事件总线（Event Bus）
- [ ] 消息队列
- [ ] 轻量级微内核

---

## 十一、安全与权限

### 11.1 安全机制
- [ ] 工作区隔离
- [ ] 路径遍历防护
- [ ] 允许列表（allowFrom）

### 11.2 凭证管理
- [ ] 环境变量
- [ ] 配置文件加密
- [ ] API Key 管理

---

## 十二、DevOps 与部署

### 12.1 容器化
- [ ] Dockerfile 编写
- [ ] Docker 运行
- [ ] 体积优化

### 12.2 进程管理
- [ ] 守护进程模式
- [ ] 信号处理
- [ ] 优雅关闭

---

## 十三、测试与质量

### 13.1 测试框架
- [ ] pytest
- [ ] pytest-asyncio
- [ ] 单元测试
- [ ] 集成测试
- [ ] 异步测试

### 13.2 代码质量
- [ ] ruff（lint + format）
- [ ] mypy 类型检查
- [ ] 代码覆盖率
- [ ] pre-commit hooks

---

## 十四、学习路径建议

### 初学者路径（按顺序）
1. Python 3.11+ 基础
2. 类型注解与 Pydantic
3. CLI 开发（Typer）
4. 异步编程与 asyncio
5. FastAPI Web 框架

### 进阶路径
1. AI/LLM 集成
2. 即时通讯平台集成
3. 架构设计模式
4. 事件驱动编程
5. 任务调度

### 专家路径
1. 插件系统设计
2. 多 Agent 架构
3. 轻量级框架设计
4. 性能优化

---

## 十五、相关学习资源

- `01-python-basics.ipynb` - Python 基础
- `02-async-python.ipynb` - 异步编程
- `03-cli-typer.ipynb` - CLI 开发
- `04-pydantic.ipynb` - 数据验证

---

*注：本知识图谱基于 nanobot 项目扫描生成，涵盖约 80+ 个知识点。建议根据实际工作需要选择性深入学习。*
