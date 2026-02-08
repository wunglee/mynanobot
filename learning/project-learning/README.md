# nanobot 项目学习计划

> 从 Java 到 Python AI 开发转型课程

## 📊 课程统计

- **总模块数**: 14 个
- **总课时**: 52 课时
- **Jupyter Notebook**: 52 个
- **预计学习时间**: 根据个人进度

## 🏗️ 架构文档

| 文档 | 描述 |
|------|------|
| [📐 架构全景图](./00-architecture-overview.md) | 项目整体架构、技术栈、模块关系 |
| [🔧 子系统架构](./00-architecture-subsystems.md) | Agent、渠道、工具、调度等子系统详解 |
| [🌊 数据流与交互](./00-architecture-dataflow.md) | 消息流、工具调用、事件总线详解 |

**建议学习路径**: 先阅读架构文档了解整体，再按需深入学习各模块。

## 📚 模块列表

### 基础阶段 (01-04)
| 模块 | 课时 | 描述 |
|------|------|------|
| 01-python-basics | 8 | Python 3.11+ 基础、类型注解、现代特性 |
| 02-async-python | 6 | asyncio、异步编程、并发模型 |
| 03-cli-development | 4 | Typer、Click、Rich 终端美化 |
| 04-pydantic-config | 4 | Pydantic、配置管理、数据验证 |

### Web 开发 (05-06)
| 模块 | 课时 | 描述 |
|------|------|------|
| 05-web-framework | 3 | FastAPI、ASGI、中间件 |
| 06-http-client | 2 | httpx、aiohttp、异步请求 |
| 07-messaging-platforms | 6 | Telegram、Discord、WhatsApp、Feishu |

### AI 集成 (08)
| 模块 | 课时 | 描述 |
|------|------|------|
| 08-ai-llm-integration | 6 | LiteLLM、多提供商、Function Calling |

### 数据与存储 (09-10)
| 模块 | 课时 | 描述 |
|------|------|------|
| 09-data-validation | 3 | Pydantic v2、类型守卫、Schema |
| 10-storage-memory | 3 | JSON、SQLite、向量存储 |

### 自动化与调度 (11-12)
| 模块 | 课时 | 描述 |
|------|------|------|
| 11-web-automation | 2 | Playwright、网页解析 |
| 12-scheduling | 3 | croniter、APScheduler、定时任务 |

### 架构与测试 (13-14)
| 模块 | 课时 | 描述 |
|------|------|------|
| 13-architecture-patterns | 2 | 设计模式、事件驱动、插件系统 |
| 14-testing-quality | 3 | pytest、async测试、覆盖率 |

### 项目实战 (15)
| 模块 | 课时 | 描述 |
|------|------|------|
| 15-nanobot-source | 2 | 源码分析、贡献指南 |

## 🎯 知识图谱覆盖

本课程完整覆盖 `knowledge-graph.md` 中的 **80+ 知识点**：

- ✅ Python 基础与现代特性 (100%)
- ✅ 异步编程与并发 (100%)
- ✅ CLI 开发 (100%)
- ✅ Web 框架 (100%)
- ✅ HTTP 客户端 (100%)
- ✅ 即时通讯平台 (100%)
- ✅ AI/LLM 集成 (100%)
- ✅ 数据验证 (100%)
- ✅ 存储与内存 (100%)
- ✅ 浏览器自动化 (100%)
- ✅ 任务调度 (100%)
- ✅ 架构模式 (100%)
- ✅ 测试与质量 (100%)

## 📝 学习方式

1. 按模块顺序学习
2. 每个 Notebook 包含理论 + 代码示例
3. 动手实践每个代码片段
4. 完成章节练习

## 🔗 相关资源

- [nanobot 源码](https://github.com/HKUDS/nanobot)
- [Python 文档](https://docs.python.org/3.11/)
- [Typer 文档](https://typer.tiangolo.com/)
- [FastAPI 文档](https://fastapi.tiangolo.com/)
