# 02 - 异步 Python 编程

> asyncio、异步 I/O、并发模型

## 📚 课时列表

| 课时 | 文件 | 主题 |
|------|------|------|
| 02-01 | `02-01-async-intro.ipynb` | 异步编程概述 |
| 02-02 | `02-02-async-await.ipynb` | async/await 基础 |
| 02-03 | `02-03-event-loop.ipynb` | 事件循环机制 |
| 02-04 | `02-04-tasks.ipynb` | Task 与并发执行 |
| 02-05 | `02-05-queues.ipynb` | 异步队列与生产者-消费者 |
| 02-06 | `02-06-streams.ipynb` | 异步流处理 |

## 🎯 学习目标

完成本模块后，你将能够：

- 理解异步编程与同步编程的区别
- 使用 async/await 编写异步代码
- 管理多个并发任务
- 使用异步队列进行数据流处理

## 🔗 相关代码

- `nanobot/agent/loop.py` - Agent 异步主循环
- `nanobot/bus/queue.py` - 异步消息队列
- `nanobot/channels/` - 各渠道的异步实现

---

*本模块对应 knowledge-graph.md 中的 "异步编程与并发"*
