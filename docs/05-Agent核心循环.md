# 第五层：Agent 核心循环

> 📌 **核心文件**：`nanobot/agent/loop.py` (~330 行)  
> 这是 nanobot 的大脑，理解这个文件就理解了整个系统的运作机制。

## 概述

`AgentLoop` 是 nanobot 最核心的组件，它实现了基于 LLM 的 **ReAct**（Reasoning + Acting）循环：

```
用户输入 → 构建上下文 → 调用 LLM → 执行工具 → 再次调用 LLM → ... → 最终响应
```

## 核心类：AgentLoop

### 类定义

```python
class AgentLoop:
    """
    The agent loop is the core processing engine.
    
    It:
    1. Receives messages from the bus
    2. Builds context with history, memory, skills
    3. Calls the LLM
    4. Executes tool calls
    5. Sends responses back
    """
```

### 初始化参数

```python
def __init__(
    self,
    bus: MessageBus,              # 消息总线
    provider: LLMProvider,        # LLM 提供商
    workspace: Path,              # 工作区路径
    model: str | None = None,     # 模型名称
    max_iterations: int = 20,     # 最大迭代次数
    brave_api_key: str | None = None  # Web 搜索 API Key
):
```

### 核心组件

AgentLoop 组合了 5 个关键组件：

```python
self.context = ContextBuilder(workspace)      # 上下文构建器
self.sessions = SessionManager(workspace)     # 会话管理器
self.tools = ToolRegistry()                   # 工具注册表
self.subagents = SubagentManager(...)         # 子代理管理器
self.bus = bus                                # 消息总线
```

## 主要方法详解

### 1. 工具注册：`_register_default_tools()`

在初始化时注册所有默认工具：

```python
def _register_default_tools(self) -> None:
    """Register the default set of tools."""
    # 文件工具
    self.tools.register(ReadFileTool())
    self.tools.register(WriteFileTool())
    self.tools.register(EditFileTool())
    self.tools.register(ListDirTool())
    
    # Shell 工具
    self.tools.register(ExecTool(working_dir=str(self.workspace)))
    
    # Web 工具
    self.tools.register(WebSearchTool(api_key=self.brave_api_key))
    self.tools.register(WebFetchTool())
    
    # 消息工具（发送到渠道）
    message_tool = MessageTool(send_callback=self.bus.publish_outbound)
    self.tools.register(message_tool)
    
    # 子代理生成工具
    spawn_tool = SpawnTool(manager=self.subagents)
    self.tools.register(spawn_tool)
```

**设计亮点**：
- 所有工具都实现了统一的 `Tool` 接口
- 通过注册表模式动态管理
- 易于扩展新工具

### 2. 主循环：`run()`

这是 Agent 的主事件循环，持续监听消息总线：

```python
async def run(self) -> None:
    """Run the agent loop, processing messages from the bus."""
    self._running = True
    logger.info("Agent loop started")
    
    while self._running:
        try:
            # 等待下一条消息（超时 1 秒）
            msg = await asyncio.wait_for(
                self.bus.consume_inbound(),
                timeout=1.0
            )
            
            # 处理消息
            try:
                response = await self._process_message(msg)
                if response:
                    await self.bus.publish_outbound(response)
            except Exception as e:
                logger.error(f"Error processing message: {e}")
                # 发送错误响应
                await self.bus.publish_outbound(OutboundMessage(
                    channel=msg.channel,
                    chat_id=msg.chat_id,
                    content=f"Sorry, I encountered an error: {str(e)}"
                ))
        except asyncio.TimeoutError:
            continue  # 超时后继续等待
```

**设计亮点**：
- 使用 `asyncio.wait_for` 实现超时机制
- 异常捕获确保单个消息失败不会导致整个循环崩溃
- 通过 `self._running` 标志实现优雅关闭

### 3. 消息处理：`_process_message()`

这是最核心的方法，实现了完整的 ReAct 循环：

```python
async def _process_message(self, msg: InboundMessage) -> OutboundMessage | None:
    """
    Process a single inbound message.
    """
    # 1. 处理系统消息（子代理公告）
    if msg.channel == "system":
        return await self._process_system_message(msg)
    
    logger.info(f"Processing message from {msg.channel}:{msg.sender_id}")
    
    # 2. 获取或创建会话
    session = self.sessions.get_or_create(msg.session_key)
    
    # 3. 更新工具上下文（告诉工具当前的 channel 和 chat_id）
    message_tool = self.tools.get("message")
    if isinstance(message_tool, MessageTool):
        message_tool.set_context(msg.channel, msg.chat_id)
    
    spawn_tool = self.tools.get("spawn")
    if isinstance(spawn_tool, SpawnTool):
        spawn_tool.set_context(msg.channel, msg.chat_id)
    
    # 4. 构建初始消息列表（包含系统提示、历史、当前消息）
    messages = self.context.build_messages(
        history=session.get_history(),
        current_message=msg.content,
        media=msg.media if msg.media else None,
    )
    
    # 5. ReAct 循环
    iteration = 0
    final_content = None
    
    while iteration < self.max_iterations:
        iteration += 1
        
        # 5a. 调用 LLM
        response = await self.provider.chat(
            messages=messages,
            tools=self.tools.get_definitions(),
            model=self.model
        )
        
        # 5b. 处理工具调用
        if response.has_tool_calls:
            # 将 LLM 的工具调用添加到消息历史
            tool_call_dicts = [
                {
                    "id": tc.id,
                    "type": "function",
                    "function": {
                        "name": tc.name,
                        "arguments": json.dumps(tc.arguments)
                    }
                }
                for tc in response.tool_calls
            ]
            messages = self.context.add_assistant_message(
                messages, response.content, tool_call_dicts
            )
            
            # 执行所有工具调用
            for tool_call in response.tool_calls:
                args_str = json.dumps(tool_call.arguments)
                logger.debug(f"Executing tool: {tool_call.name} with arguments: {args_str}")
                result = await self.tools.execute(tool_call.name, tool_call.arguments)
                
                # 将工具结果添加到消息历史
                messages = self.context.add_tool_result(
                    messages, tool_call.id, tool_call.name, result
                )
        else:
            # 没有工具调用，得到最终响应
            final_content = response.content
            break
    
    # 6. 如果达到最大迭代次数仍未结束
    if final_content is None:
        final_content = "I've completed processing but have no response to give."
    
    # 7. 保存到会话历史
    session.add_message("user", msg.content)
    session.add_message("assistant", final_content)
    self.sessions.save(session)
    
    # 8. 返回响应
    return OutboundMessage(
        channel=msg.channel,
        chat_id=msg.chat_id,
        content=final_content
    )
```

## ReAct 循环详解

这是 nanobot 的核心算法，让我们分解一下：

### 阶段 1：上下文构建

```python
messages = self.context.build_messages(
    history=session.get_history(),
    current_message=msg.content,
)
```

生成的 `messages` 结构：
```json
[
  {
    "role": "system",
    "content": "# nanobot 🐈\nYou are nanobot...\n\n# Memory\n...\n\n# Skills\n..."
  },
  {
    "role": "user",
    "content": "之前的用户消息"
  },
  {
    "role": "assistant",
    "content": "之前的助手回复"
  },
  {
    "role": "user",
    "content": "当前用户消息"
  }
]
```

### 阶段 2：LLM 推理

```python
response = await self.provider.chat(
    messages=messages,
    tools=self.tools.get_definitions(),
    model=self.model
)
```

LLM 可能返回：
- **文本响应**：直接回复用户
- **工具调用**：需要执行某些操作后再回复

### 阶段 3：工具执行

如果 LLM 决定使用工具：

```python
if response.has_tool_calls:
    for tool_call in response.tool_calls:
        # 执行工具
        result = await self.tools.execute(
            tool_call.name, 
            tool_call.arguments
        )
        
        # 将结果添加到对话历史
        messages = self.context.add_tool_result(
            messages, tool_call.id, tool_call.name, result
        )
```

### 阶段 4：迭代或结束

- **有工具调用**：继续循环，让 LLM 看到工具结果后再次推理
- **无工具调用**：结束循环，返回最终响应

### 完整流程示例

用户：**"读取 config.json 文件的内容并总结"**

```
迭代 1:
  LLM → 调用 read_file(path="config.json")
  Tool → 返回文件内容
  
迭代 2:
  LLM → 看到文件内容后，总结配置
  返回 → "这个配置文件包含了..."
```

## 系统消息处理：`_process_system_message()`

用于处理子代理的公告消息：

```python
async def _process_system_message(self, msg: InboundMessage) -> OutboundMessage | None:
    """
    Process a system message (e.g., subagent announce).
    
    The chat_id field contains "original_channel:original_chat_id" to route
    the response back to the correct destination.
    """
    # 解析原始来源
    if ":" in msg.chat_id:
        parts = msg.chat_id.split(":", 1)
        origin_channel = parts[0]
        origin_chat_id = parts[1]
    else:
        origin_channel = "cli"
        origin_chat_id = msg.chat_id
    
    # 使用原始会话的上下文
    session_key = f"{origin_channel}:{origin_chat_id}"
    session = self.sessions.get_or_create(session_key)
    
    # ... 类似的 ReAct 循环 ...
    
    # 路由回原始渠道
    return OutboundMessage(
        channel=origin_channel,
        chat_id=origin_chat_id,
        content=final_content
    )
```

**设计亮点**：
- 子代理完成后可以主动通知用户
- 通过 `chat_id` 编码原始来源信息
- 响应会路由回正确的渠道

## 直接处理：`process_direct()`

提供同步 API 供 CLI 直接调用：

```python
async def process_direct(self, content: str, session_key: str = "cli:direct") -> str:
    """
    Process a message directly (for CLI usage).
    """
    msg = InboundMessage(
        channel="cli",
        sender_id="user",
        chat_id="direct",
        content=content
    )
    
    response = await self._process_message(msg)
    return response.content if response else ""
```

## 关键设计决策

### 1. 为什么使用迭代次数限制？

```python
max_iterations: int = 20
```

防止 LLM 陷入无限循环：
- LLM 可能一直调用工具
- 某些工具执行失败后 LLM 可能重试
- 限制迭代次数确保最终能给出响应

### 2. 为什么工具需要 `set_context()`？

```python
message_tool.set_context(msg.channel, msg.chat_id)
spawn_tool.set_context(msg.channel, msg.chat_id)
```

某些工具需要知道当前对话的来源：
- `message` 工具需要知道往哪个渠道发送消息
- `spawn` 工具需要知道子代理完成后通知谁

### 3. 为什么要保存会话历史？

```python
session.add_message("user", msg.content)
session.add_message("assistant", final_content)
self.sessions.save(session)
```

- 多轮对话需要上下文
- 用户可能引用之前的内容
- 持久化到磁盘避免重启后丢失

## 性能优化

### 1. 异步执行

所有 I/O 操作都是异步的：
```python
await self.provider.chat(...)       # 网络请求
await self.tools.execute(...)       # 可能的文件/网络操作
await self.bus.publish_outbound(...) # 队列操作
```

### 2. 超时机制

避免长时间阻塞：
```python
msg = await asyncio.wait_for(
    self.bus.consume_inbound(),
    timeout=1.0
)
```

### 3. 并发处理

主循环可以与其他服务并发运行：
```python
await asyncio.gather(
    agent.run(),              # Agent 循环
    bus.dispatch_outbound(),  # 消息分发
    channel_manager.start(),  # 渠道监听
)
```

## 错误处理

### 1. 消息处理错误

```python
try:
    response = await self._process_message(msg)
    await self.bus.publish_outbound(response)
except Exception as e:
    logger.error(f"Error processing message: {e}")
    # 发送友好的错误消息给用户
    await self.bus.publish_outbound(OutboundMessage(
        channel=msg.channel,
        chat_id=msg.chat_id,
        content=f"Sorry, I encountered an error: {str(e)}"
    ))
```

### 2. LLM 调用错误

在 `LiteLLMProvider` 中处理：
```python
try:
    response = await acompletion(**kwargs)
    return self._parse_response(response)
except Exception as e:
    return LLMResponse(
        content=f"Error calling LLM: {str(e)}",
        finish_reason="error",
    )
```

## 扩展点

### 1. 自定义工具

继承 `Tool` 基类并注册：
```python
class MyCustomTool(Tool):
    @property
    def name(self) -> str:
        return "my_tool"
    
    async def execute(self, **kwargs) -> str:
        # 实现你的逻辑
        return "result"

# 注册
agent.tools.register(MyCustomTool())
```

### 2. 自定义消息处理

可以继承 `AgentLoop` 并覆盖 `_process_message()`：
```python
class CustomAgentLoop(AgentLoop):
    async def _process_message(self, msg: InboundMessage):
        # 添加自定义逻辑
        if msg.content.startswith("/custom"):
            return OutboundMessage(...)
        
        # 调用父类方法
        return await super()._process_message(msg)
```

## 调试技巧

### 1. 查看完整的消息流

在关键位置添加日志：
```python
logger.debug(f"Messages before LLM call: {json.dumps(messages, indent=2)}")
logger.debug(f"LLM response: {response}")
logger.debug(f"Tool result: {result}")
```

### 2. 检查工具定义

```python
tools = agent.tools.get_definitions()
print(json.dumps(tools, indent=2))
```

### 3. 查看会话历史

```python
session = agent.sessions.get_or_create("cli:default")
print(session.get_history())
```

## 小结

通过本章，你应该掌握了：
- ✅ AgentLoop 的核心结构和职责
- ✅ ReAct 循环的完整流程
- ✅ 工具注册和执行机制
- ✅ 会话管理和消息路由
- ✅ 错误处理和性能优化

**关键要点**：
- AgentLoop 是整个系统的大脑
- ReAct 循环是核心算法（推理 → 行动 → 推理...）
- 工具是 Agent 与外界交互的唯一方式
- 异步设计确保高性能

**下一步**：[06-上下文构建.md](./06-上下文构建.md) - 了解如何组装 Prompt。
