# nanobot 数据流与交互详解

> 深入解析 nanobot 中的消息流、工具调用、WebSocket 通信等核心交互机制。

## 📋 目录

1. [消息生命周期](#消息生命周期)
2. [Agent 对话流](#agent-对话流)
3. [工具调用流](#工具调用流)
4. [事件传播机制](#事件传播机制)
5. [状态管理](#状态管理)
6. [错误处理流](#错误处理流)

---

## 消息生命周期

### 完整消息流转

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Message Lifecycle                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Step 1: 接收 (Receive)                                                     │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                               │
│  │ Telegram│────▶│ Handler │────▶│  Parse  │                               │
│  │  Update │     │  (Raw)  │     │ Message │                               │
│  └─────────┘     └─────────┘     └────┬────┘                               │
│                                        │                                    │
│  Step 2: 路由 (Route)                   │                                    │
│                                        ▼                                    │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                               │
│  │ Session │◀────│  Event  │◀────│  Event  │                               │
│  │ Manager │     │   Bus   │     │  Emit   │                               │
│  └────┬────┘     └─────────┘     └─────────┘                               │
│       │                                                                     │
│  Step 3: 处理 (Process)                 │                                    │
│       │                                 ▼                                    │
│  ┌────┴────┐     ┌─────────┐     ┌─────────┐                               │
│  │  Agent  │◀────│ Context │◀────│  Build  │                               │
│  │  Loop   │     │ Builder │     │ Prompt  │                               │
│  └────┬────┘     └─────────┘     └─────────┘                               │
│       │                                                                     │
│  Step 4: 响应 (Respond)                 │                                    │
│       │                                 ▼                                    │
│  ┌────┴────┐     ┌─────────┐     ┌─────────┐                               │
│  │ Channel │◀────│ Format  │◀────│  LLM    │                               │
│  │  Send   │     │ Output  │     │ Response│                               │
│  └─────────┘     └─────────┘     └─────────┘                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 消息数据结构

```python
# 标准消息格式
class Message:
    """标准消息结构"""
    
    # 元信息
    id: str                    # 消息唯一 ID
    timestamp: datetime        # 时间戳
    
    # 渠道信息
    channel: str               # 渠道类型: telegram/discord/whatsapp/feishu
    channel_id: str           # 渠道实例 ID
    
    # 会话信息
    session_id: str           # 会话 ID
    chat_id: str              # 聊天/群组 ID
    chat_type: str            # private/group/channel
    
    # 用户信息
    user_id: str              # 用户 ID
    user_name: str            # 用户名称
    
    # 内容
    content_type: str         # text/image/voice/file
    text: Optional[str]       # 文本内容
    file_url: Optional[str]   # 文件 URL
    
    # 上下文
    reply_to: Optional[str]   # 回复的消息 ID
    mentioned: bool           # 是否被 @
    
    # 原始数据
    raw: dict                 # 原始渠道数据
```

### Telegram 消息流转示例

```python
# Step 1: 接收
async def handle_telegram_update(update: Update, context):
    """处理 Telegram 更新"""
    
    # 转换为标准消息格式
    message = Message(
        id=str(update.message.message_id),
        timestamp=datetime.fromtimestamp(update.message.date),
        channel="telegram",
        channel_id=f"telegram:{update.effective_user.id}",
        session_id=f"telegram:{update.effective_chat.id}",
        chat_id=str(update.effective_chat.id),
        chat_type="private" if update.effective_chat.type == "private" else "group",
        user_id=str(update.effective_user.id),
        user_name=update.effective_user.username or update.effective_user.first_name,
        content_type="text",
        text=update.message.text,
        reply_to=str(update.message.reply_to_message.message_id) if update.message.reply_to_message else None,
        mentioned=bot_username in update.message.text if update.message.text else False,
        raw=update.to_dict()
    )

# Step 2: 路由
async def route_message(message: Message):
    """路由消息到处理器"""
    
    # 权限检查
    if not await check_permission(message):
        return
    
    # 发布事件
    await event_bus.emit("message.received", {
        "message": message,
        "timestamp": datetime.now()
    })
    
    # 获取或创建会话
    session = await session_manager.get_or_create(message.session_id)
    
    # 添加到会话历史
    await session.add_message({
        "role": "user",
        "content": message.text
    })

# Step 3: 处理
async def process_message(session: Session, message: Message):
    """处理消息"""
    
    # 构建上下文
    context = await context_builder.build(
        session=session,
        new_message=message.text
    )
    
    # Agent 处理
    async for chunk in agent_loop.run(
        context=context,
        session_id=session.id
    ):
        # 流式输出到会话
        await session.send_stream(chunk)

# Step 4: 响应
async def send_response(session: Session, response: str):
    """发送响应"""
    
    # 根据渠道类型发送
    channel = channel_manager.get(session.channel_type)
    
    # 分块发送长消息
    chunks = split_message(response, max_length=4096)
    for chunk in chunks:
        await channel.send_message(
            chat_id=session.chat_id,
            text=chunk
        )
    
    # 添加到会话历史
    await session.add_message({
        "role": "assistant",
        "content": response
    })
```

---

## Agent 对话流

### 对话循环详解

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Agent Conversation Loop                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────┐                                                                │
│  │  Start  │                                                                │
│  └────┬────┘                                                                │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Step 1: Build Context                                             │   │
│  │                                                                     │   │
│  │  messages = [                                                       │   │
│  │    {"role": "system", "content": system_prompt},                    │   │
│  │    {"role": "user", "content": "Previous message..."},              │   │
│  │    {"role": "assistant", "content": "..."},                         │   │
│  │    {"role": "user", "content": current_message},                    │   │
│  │  ]                                                                  │   │
│  │                                                                     │   │
│  │  + Memory Retrieval                                                 │   │
│  │  + Skill Context                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Step 2: LLM Call                                                  │   │
│  │                                                                     │   │
│  │  response = await litellm.acompletion(                              │   │
│  │    model="anthropic/claude-3-opus",                                 │   │
│  │    messages=messages,                                               │   │
│  │    tools=available_tools,                                           │   │
│  │    stream=True                                                      │   │
│  │  )                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Step 3: Parse Response                                            │   │
│  │                                                                     │   │
│  │  if response.has_tool_calls:                                        │   │
│  │    tool_calls = parse_tool_calls(response)                          │   │
│  │    ───────▶ Execute Tools ───────▶ Loop back                        │   │
│  │  else:                                                              │   │
│  │    content = response.content                                       │   │
│  │    ───────▶ Return to User                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────┐    Yes    ┌─────────┐                                         │
│  │ Tool    │──────────▶│ Tool    │                                         │
│  │ Calls?  │           │ Execute │                                         │
│  └────┬────┘           └────┬────┘                                         │
│       │ No                  │                                              │
│       │                     │                                              │
│       ▼                     │                                              │
│  ┌─────────┐                │                                              │
│  │  Return │◀───────────────┘                                              │
│  │ Response│                                                               │
│  └─────────┘                                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 流式响应处理

```python
# nanobot/agent/loop.py
async def run_streaming(
    self,
    messages: list[dict],
    model: str,
    tools: list[dict]
) -> AsyncGenerator[StreamChunk, None]:
    """流式运行 Agent"""
    
    # 累积状态
    content_buffer = ""
    tool_calls_buffer: list[ToolCall] = []
    current_tool_call: Optional[ToolCall] = None
    
    # 调用 LLM（流式）
    stream = await self.provider.chat_completion(
        messages=messages,
        model=model,
        tools=tools if tools else None,
        stream=True
    )
    
    async for chunk in stream:
        # 处理内容
        if chunk.choices[0].delta.content:
            text = chunk.choices[0].delta.content
            content_buffer += text
            yield StreamChunk(type="text", content=text)
        
        # 处理工具调用增量
        if chunk.choices[0].delta.tool_calls:
            for tool_delta in chunk.choices[0].delta.tool_calls:
                if tool_delta.index >= len(tool_calls_buffer):
                    # 新工具调用
                    tool_calls_buffer.append(ToolCall(
                        id=tool_delta.id,
                        name=tool_delta.function.name,
                        arguments=""
                    ))
                
                # 累积参数
                if tool_delta.function.arguments:
                    tool_calls_buffer[tool_delta.index].arguments += tool_delta.function.arguments
    
    # 如果有工具调用，执行它们
    if tool_calls_buffer:
        yield StreamChunk(type="tool_start", content="")
        
        for tool_call in tool_calls_buffer:
            yield StreamChunk(
                type="tool_call",
                content=f"Using tool: {tool_call.name}"
            )
            
            # 执行工具
            result = await self.tools.execute(
                name=tool_call.name,
                arguments=json.loads(tool_call.arguments)
            )
            
            yield StreamChunk(
                type="tool_result",
                content=result
            )
        
        yield StreamChunk(type="tool_end", content="")
```

---

## 工具调用流

### 工具执行流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Tool Invocation Flow                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Tool Definition                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  {                                                                  │   │
│  │    "type": "function",                                              │   │
│  │    "function": {                                                    │   │
│  │      "name": "read_file",                                           │   │
│  │      "description": "Read a file from workspace",                   │   │
│  │      "parameters": {                                                │   │
│  │        "type": "object",                                            │   │
│  │        "properties": {                                              │   │
│  │          "path": {"type": "string"}                                 │   │
│  │        },                                                           │   │
│  │        "required": ["path"]                                         │   │
│  │      }                                                              │   │
│  │    }                                                                │   │
│  │  }                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  2. LLM Request with Tools                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  messages = [                                                       │   │
│  │    {"role": "system", "content": "You have access to tools..."},    │   │
│  │    {"role": "user", "content": "Read the README"}                   │   │
│  │  ]                                                                  │   │
│  │                                                                     │   │
│  │  tools = [read_file_schema, write_file_schema, ...]                 │   │
│  │                                                                     │   │
│  │  response = llm(messages, tools=tools)                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  3. LLM Response with Tool Calls                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  {                                                                  │   │
│  │    "content": null,                                                 │   │
│  │    "tool_calls": [{                                                 │   │
│  │      "id": "call_123",                                              │   │
│  │      "type": "function",                                            │   │
│  │      "function": {                                                  │   │
│  │        "name": "read_file",                                         │   │
│  │        "arguments": "{\"path\": \"README.md\"}"                       │   │
│  │      }                                                              │   │
│  │    }]                                                               │   │
│  │  }                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  4. Tool Execution                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐                      │   │
│  │  │  Parse   │───▶│ Validate │───▶│  Check   │                      │   │
│  │  │   Args   │    │  Schema  │    │ Security │                      │   │
│  │  └──────────┘    └──────────┘    └────┬─────┘                      │   │
│  │                                        │                            │   │
│  │                                        ▼                            │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐                      │   │
│  │  │  Format  │◀───│  Execute │◀───│ Dispatch │                      │   │
│  │  │  Result  │    │   Tool   │    │   Tool   │                      │   │
│  │  └──────────┘    └──────────┘    └──────────┘                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  5. Tool Result to LLM                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  messages.append({                                                  │   │
│  │    "role": "assistant",                                             │   │
│  │    "content": None,                                                 │   │
│  │    "tool_calls": [tool_call]                                        │   │
│  │  })                                                                 │   │
│  │  messages.append({                                                  │   │
│  │    "role": "tool",                                                  │   │
│  │    "tool_call_id": "call_123",                                      │   │
│  │    "content": "# README\n\nProject description..."                   │   │
│  │  })                                                                 │   │
│  │                                                                     │   │
│  │  # Send back to LLM for final response                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 工具注册与执行

```python
# nanobot/agent/tools/registry.py
class ToolRegistry:
    """工具注册表"""
    
    def __init__(self):
        self._tools: dict[str, Tool] = {}
    
    def register(self, tool: Tool):
        """注册工具"""
        self._tools[tool.name] = tool
        logger.info(f"Registered tool: {tool.name}")
    
    def get_schemas(self) -> list[dict]:
        """获取工具 schemas（供 LLM 使用）"""
        return [
            {
                "type": "function",
                "function": {
                    "name": tool.name,
                    "description": tool.description,
                    "parameters": tool.parameters
                }
            }
            for tool in self._tools.values()
        ]
    
    async def execute(self, name: str, arguments: dict) -> str:
        """执行工具"""
        tool = self._tools.get(name)
        if not tool:
            raise ToolNotFoundError(f"Tool not found: {name}")
        
        # 验证参数
        validated = self._validate_args(tool, arguments)
        
        # 执行
        start_time = time.time()
        try:
            result = await tool.execute(**validated)
            duration = time.time() - start_time
            
            logger.info(f"Tool {name} executed in {duration:.2f}s")
            return str(result)
            
        except Exception as e:
            logger.error(f"Tool {name} failed: {e}")
            return f"Error: {str(e)}"
```

---

## 事件传播机制

### 事件总线架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Event Bus Architecture                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      Event Types                                    │   │
│  │                                                                     │   │
│  │  Core Events:                                                       │   │
│  │  ├── message.received    # 新消息到达                              │   │
│  │  ├── message.sent        # 消息已发送                              │   │
│  │  ├── session.started     # 新会话开始                              │   │
│  │  ├── session.ended       # 会话结束                                │   │
│  │  ├── tool.invoking       # 工具即将调用                            │   │
│  │  ├── tool.completed      # 工具执行完成                            │   │
│  │  ├── cron.triggered      # 定时任务触发                            │   │
│  │  └── error.occurred      # 发生错误                                │   │
│  │                                                                     │   │
│  │  Channel Events:                                                    │   │
│  │  ├── channel.connected   # 渠道连接成功                            │   │
│  │  ├── channel.disconnected# 渠道断开                                │   │
│  │  └── channel.error       # 渠道错误                                │   │
│  │                                                                     │   │
│  │  Agent Events:                                                      │   │
│  │  ├── agent.thinking      # Agent 开始思考                          │   │
│  │  ├── agent.responded     # Agent 响应完成                          │   │
│  │  └── agent.error         # Agent 错误                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Event Flow                                       │   │
│  │                                                                     │   │
│  │   Publisher                    Event Bus                  Subscribers│   │
│  │                                                                     │   │
│  │  ┌──────────┐                ┌──────────┐              ┌──────────┐  │   │
│  │  │ Telegram │───────────────▶│          │─────────────▶│  Agent   │  │   │
│  │  │ Handler  │   message.     │          │              │  Loop    │  │   │
│  │  └──────────┘   received     │          │              └──────────┘  │   │
│  │                              │          │                           │   │
│  │  ┌──────────┐                │          │              ┌──────────┐  │   │
│  │  │  Agent   │───────────────▶│          │─────────────▶│  Logger  │  │   │
│  │  │  Loop    │   tool.        │          │              └──────────┘  │   │
│  │  └──────────┘   completed    │          │                           │   │
│  │                              │          │              ┌──────────┐  │   │
│  │  ┌──────────┐                │          │─────────────▶│  Memory  │  │   │
│  │  │  Cron    │───────────────▶│          │              │  Store   │  │   │
│  │  │ Service  │   cron.        │          │              └──────────┘  │   │
│  │  └──────────┘   triggered    └──────────┘                           │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 事件处理实现

```python
# nanobot/bus/events.py
from typing import Callable, Awaitable, Any
from dataclasses import dataclass
from datetime import datetime
import asyncio

@dataclass
class Event:
    """事件对象"""
    type: str
    data: dict
    timestamp: datetime
    source: str

EventHandler = Callable[[Event], Awaitable[None]]

class EventBus:
    """异步事件总线"""
    
    def __init__(self):
        self._handlers: dict[str, list[EventHandler]] = {}
        self._middleware: list[Callable] = []
    
    def on(self, event_type: str, handler: EventHandler):
        """订阅事件"""
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)
    
    def off(self, event_type: str, handler: EventHandler):
        """取消订阅"""
        if event_type in self._handlers:
            self._handlers[event_type].remove(handler)
    
    def use(self, middleware: Callable):
        """添加中间件"""
        self._middleware.append(middleware)
    
    async def emit(self, event_type: str, data: dict, source: str = ""):
        """发布事件"""
        event = Event(
            type=event_type,
            data=data,
            timestamp=datetime.now(),
            source=source
        )
        
        # 执行中间件
        for middleware in self._middleware:
            event = await middleware(event)
            if event is None:  # 中间件可取消事件
                return
        
        # 执行处理器
        handlers = self._handlers.get(event_type, [])
        
        if not handlers:
            return
        
        # 并发执行（带错误隔离）
        results = await asyncio.gather(
            *[self._run_handler(handler, event) for handler in handlers],
            return_exceptions=True
        )
        
        # 记录错误
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                logger.error(f"Event handler {i} failed: {result}")
    
    async def _run_handler(self, handler: EventHandler, event: Event):
        """运行单个处理器（带超时）"""
        try:
            await asyncio.wait_for(
                handler(event),
                timeout=30.0
            )
        except asyncio.TimeoutError:
            logger.warning(f"Event handler timed out: {handler}")
        except Exception as e:
            logger.error(f"Event handler error: {e}")
            raise


# 使用示例
# 订阅事件
event_bus.on("message.received", async (event) => {
    logger.info(f"New message: {event.data['message'].text}")
})

# 发布事件
await event_bus.emit(
    "message.received",
    {"message": message},
    source="telegram"
)
```

---

## 状态管理

### 会话状态

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Session State Management                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Session Structure                                │   │
│  │                                                                     │   │
│  │  session_id: str          # 唯一标识符                             │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │  Metadata                                                   │   │   │
│  │  │  ├── channel: str          # 渠道类型                        │   │   │
│  │  │  ├── chat_id: str          # 聊天 ID                         │   │   │
│  │  │  ├── user_id: str          # 用户 ID                         │   │   │
│  │  │  ├── created_at: datetime  # 创建时间                        │   │   │
│  │  │  └── last_active: datetime # 最后活跃                        │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │  Conversation History (in-memory)                           │   │   │
│  │  │  [                                                          │   │   │
│  │  │    {"role": "user", "content": "..."},                       │   │   │
│  │  │    {"role": "assistant", "content": "..."},                  │   │   │
│  │  │    {"role": "user", "content": "..."},                       │   │   │
│  │  │    ...                                                     │   │   │
│  │  │  ]                                                          │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │  Memory (persistent)                                        │   │   │
│  │  │  ├── important_facts: list[str]   # 重要事实                │   │   │
│  │  │  ├── user_preferences: dict       # 用户偏好                │   │   │
│  │  │  └── context_summary: str         # 上下文摘要              │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    State Transitions                                │   │
│  │                                                                     │   │
│  │  IDLE ──▶ ACTIVE ──▶ PROCESSING ──▶ RESPONDING ──▶ IDLE            │   │
│  │           │           │             │                               │   │
│  │           │           │             └── 发送响应                    │   │
│  │           │           └── 执行工具/调用 LLM                        │   │
│  │           └── 收到消息                                              │   │
│  │                                                                     │   │
│  │  ERROR ──▶ IDLE (recover)                                          │   │
│  │  TIMEOUT ──▶ IDLE                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 会话管理实现

```python
# nanobot/session/manager.py
class Session:
    """会话对象"""
    
    def __init__(self, session_id: str, channel: str, chat_id: str):
        self.id = session_id
        self.channel = channel
        self.chat_id = chat_id
        self.user_id: Optional[str] = None
        
        self.messages: list[dict] = []
        self.metadata: dict = {}
        
        self.created_at = datetime.now()
        self.last_active = datetime.now()
        
        self.state: SessionState = SessionState.IDLE
        self._lock = asyncio.Lock()
    
    async def add_message(self, role: str, content: str, **kwargs):
        """添加消息到历史"""
        async with self._lock:
            self.messages.append({
                "role": role,
                "content": content,
                "timestamp": datetime.now().isoformat(),
                **kwargs
            })
            
            # 限制历史长度
            if len(self.messages) > 100:
                self.messages = self.messages[-50:]  # 保留最近 50 条
            
            self.last_active = datetime.now()
    
    def get_context(self, limit: int = 20) -> list[dict]:
        """获取对话上下文"""
        return [
            {"role": m["role"], "content": m["content"]}
            for m in self.messages[-limit:]
        ]
    
    def update_state(self, state: SessionState):
        """更新状态"""
        old_state = self.state
        self.state = state
        logger.debug(f"Session {self.id}: {old_state} -> {state}")


class SessionManager:
    """会话管理器"""
    
    def __init__(self):
        self._sessions: dict[str, Session] = {}
        self._cleanup_task: Optional[asyncio.Task] = None
    
    async def start(self):
        """启动管理器"""
        self._cleanup_task = asyncio.create_task(self._cleanup_loop())
    
    async def stop(self):
        """停止管理器"""
        if self._cleanup_task:
            self._cleanup_task.cancel()
    
    def get_or_create(
        self,
        session_id: str,
        channel: str = "",
        chat_id: str = ""
    ) -> Session:
        """获取或创建会话"""
        if session_id not in self._sessions:
            self._sessions[session_id] = Session(
                session_id=session_id,
                channel=channel,
                chat_id=chat_id
            )
        
        return self._sessions[session_id]
    
    def get(self, session_id: str) -> Optional[Session]:
        """获取会话"""
        return self._sessions.get(session_id)
    
    async def _cleanup_loop(self):
        """清理过期会话"""
        while True:
            await asyncio.sleep(300)  # 每 5 分钟
            
            now = datetime.now()
            expired = [
                sid for sid, session in self._sessions.items()
                if (now - session.last_active).seconds > 3600  # 1 小时过期
            ]
            
            for sid in expired:
                del self._sessions[sid]
                logger.info(f"Cleaned up expired session: {sid}")
```

---

## 错误处理流

### 错误处理架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Error Handling Flow                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Error Types                                      │   │
│  │                                                                     │   │
│  │  1. Channel Errors                                                  │   │
│  │     ├── ConnectionError    # 连接失败                              │   │
│  │     ├── AuthenticationError# 认证失败                              │   │
│  │     ├── RateLimitError     # 限流                                  │   │
│  │     └── TimeoutError       # 超时                                  │   │
│  │                                                                     │   │
│  │  2. LLM Errors                                                      │   │
│  │     ├── APIError           # API 调用失败                          │   │
│  │     ├── ModelNotFoundError # 模型不存在                            │   │
│  │     ├── ContextLengthError # 上下文过长                            │   │
│  │     └── ContentFilterError # 内容被过滤                            │   │
│  │                                                                     │   │
│  │  3. Tool Errors                                                     │   │
│  │     ├── ToolNotFoundError  # 工具不存在                            │   │
│  │     ├── ValidationError    # 参数验证失败                          │   │
│  │     ├── ExecutionError     # 执行失败                              │   │
│  │     └── SecurityError      # 安全限制                              │   │
│  │                                                                     │   │
│  │  4. System Errors                                                   │   │
│  │     ├── ConfigError        # 配置错误                              │   │
│  │     ├── StorageError       # 存储错误                              │   │
│  │     └── UnknownError       # 未知错误                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Error Handling Strategy                          │   │
│  │                                                                     │   │
│  │  Try ──▶ Catch ──▶ Classify ──▶ Handle ──▶ Respond                  │   │
│  │                                                                     │   │
│  │  1. Try: 执行业务逻辑                                               │   │
│  │  2. Catch: 捕获异常                                                 │   │
│  │  3. Classify: 识别错误类型                                          │   │
│  │  4. Handle: 根据类型处理                                            │   │
│  │  5. Respond: 向用户返回友好信息                                     │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 错误处理实现

```python
# nanobot/infra/errors.py
class NanobotError(Exception):
    """基础错误类"""
    
    def __init__(self, message: str, code: str = "", details: dict = None):
        super().__init__(message)
        self.message = message
        self.code = code
        self.details = details or {}

class ChannelError(NanobotError):
    """渠道错误"""
    pass

class LLMError(NanobotError):
    """LLM 错误"""
    pass

class ToolError(NanobotError):
    """工具错误"""
    pass

class SecurityError(NanobotError):
    """安全错误"""
    pass


# 错误处理器
class ErrorHandler:
    """全局错误处理器"""
    
    def __init__(self, event_bus: EventBus):
        self.event_bus = event_bus
    
    async def handle(self, error: Exception, context: dict = None) -> str:
        """处理错误并返回用户友好的消息"""
        
        # 发布错误事件
        await self.event_bus.emit("error.occurred", {
            "error_type": type(error).__name__,
            "error_message": str(error),
            "context": context
        })
        
        # 分类处理
        if isinstance(error, SecurityError):
            return self._handle_security_error(error)
        
        elif isinstance(error, LLMError):
            return self._handle_llm_error(error)
        
        elif isinstance(error, ToolError):
            return self._handle_tool_error(error)
        
        elif isinstance(error, ChannelError):
            return self._handle_channel_error(error)
        
        else:
            # 未知错误
            logger.exception("Unexpected error")
            return "抱歉，发生了意外错误。请稍后再试。"
    
    def _handle_security_error(self, error: SecurityError) -> str:
        """处理安全错误"""
        return f"⛔ 操作被安全策略阻止: {error.message}"
    
    def _handle_llm_error(self, error: LLMError) -> str:
        """处理 LLM 错误"""
        if "rate limit" in error.message.lower():
            return "⏳ 请求过于频繁，请稍后再试。"
        elif "context" in error.message.lower():
            return "📚 对话太长，请开始新会话。"
        else:
            return f"🤖 AI 服务暂时不可用: {error.message}"
    
    def _handle_tool_error(self, error: ToolError) -> str:
        """处理工具错误"""
        return f"🔧 工具执行失败: {error.message}"
    
    def _handle_channel_error(self, error: ChannelError) -> str:
        """处理渠道错误"""
        return f"📡 消息发送失败: {error.message}"


# 使用装饰器自动处理错误
def with_error_handler(handler: ErrorHandler):
    """错误处理装饰器"""
    def decorator(func):
        async def wrapper(*args, **kwargs):
            try:
                return await func(*args, **kwargs)
            except Exception as e:
                context = {
                    "function": func.__name__,
                    "args": args,
                    "kwargs": kwargs
                }
                return await handler.handle(e, context)
        return wrapper
    return decorator


# 使用示例
class AgentLoop:
    def __init__(self, error_handler: ErrorHandler):
        self.error_handler = error_handler
    
    @with_error_handler(error_handler)
    async def run(self, message: str) -> str:
        """运行 Agent（自动错误处理）"""
        # ... 业务逻辑
        pass
```

---

## 总结

nanobot 的数据流设计遵循以下原则：

1. **标准化消息格式** - 所有渠道统一转换为 Message 对象
2. **流式处理** - Agent 响应采用流式输出，提升用户体验
3. **事件驱动** - 组件间通过 Event Bus 松耦合通信
4. **状态隔离** - 每个会话独立管理状态和上下文
5. **错误隔离** - 错误处理不中断主流程，用户友好提示

这些设计保证了系统的**可扩展性**、**可维护性**和**可靠性**。
