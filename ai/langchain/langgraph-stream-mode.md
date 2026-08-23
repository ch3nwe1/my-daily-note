# LangGraph Stream Mode 详解

## 概述

LangGraph 的 `stream()` 方法支持多种流式模式（stream_mode），控制数据如何从图中流式输出。

## 所有 Stream Mode 汇总

| 模式 | 说明 | 粒度 | 需要 checkpointer |
|------|------|------|------------------|
| `values` | 每个节点执行后的**完整状态快照** | 节点级 | ❌ |
| `updates` | 每个节点执行后的**状态增量** | 节点级 | ❌ |
| `messages` | LLM 的 **token 流** | token 级 | ❌ |
| `custom` | 通过 `get_stream_writer` 发送的**自定义数据** | 自定义 | ❌ |
| `checkpoints` | **检查点事件**（格式同 `get_state()`） | 节点级 | ✅ |
| `tasks` | **任务开始/结束事件**，包含结果和错误 | 节点级 | ✅ |
| `debug` | **所有信息** = checkpoints + tasks + 额外元数据 | 最细 | ✅ |

## 1. stream_mode="values"

每个节点执行完后，返回**完整的 state 快照**（包含所有历史消息）。

```python
for state in agent.stream(inputs, stream_mode="values"):
    messages = state["messages"]        # 完整的消息列表
    last_msg = messages[-1]             # 最后一条消息
    print(f"当前共 {len(messages)} 条消息")
```

**特点**：
- 包含 HumanMessage（对话历史）
- 每次返回完整数组，越来越大
- 适合调试、追踪状态变化

## 2. stream_mode="updates"

每个节点执行完后，只返回**增量更新**（新增/变化的部分）。

```python
for update in agent.stream(inputs, stream_mode="updates"):
    for node_name, node_update in update.items():
        new_messages = node_update.get("messages", [])
        print(f"节点 [{node_name}] 新增了 {len(new_messages)} 条消息")
```

**特点**：
- 不包含 HumanMessage（因为它不是节点产出的）
- 只返回变化的字段，更省带宽
- 适合追踪"哪个节点改了什么"

## 3. stream_mode="messages"

逐 token 流式返回 LLM 生成的消息片段。

```python
for chunk in agent.stream(inputs, stream_mode="messages"):
    msg, metadata = chunk["data"]
    node = metadata["langgraph_node"]
    
    for block in msg.content_blocks:
        block_type = block.get("type")
        
        if block_type == "text":
            print(block["text"], end="")
            
        elif block_type == "tool_call_chunk":
            if block.get("name"):
                print(f"\n[调用工具: {block['name']}]")
            if block.get("args"):
                print(f"  参数片段: {block['args']}")
```

**content_blocks 的 type**：

| type | 说明 |
|------|------|
| `text` | 普通文本 |
| `tool_call_chunk` | 工具调用片段（流式） |
| `reasoning` | 推理/思考内容（reasoning 模型） |
| `image` | 图片内容 |
| `citation` | 引用 |

**特点**：
- 只返回 AIMessageChunk，不包含 HumanMessage
- 高频输出（每个 token 一次）
- 适合聊天 UI 逐字显示

**Reasoning 模型的空 chunk**：
- `gpt-5-nano` 等 reasoning 模型在思考时会产生大量 `content: []`
- 这是 thinking tokens，内容被隐藏，不在 content_blocks 中

## 4. stream_mode="custom"

允许自定义流式输出内容。

```python
from langgraph.config import get_stream_writer

@tool
def search(query: str) -> str:
    writer = get_stream_writer()      # 注意：不是参数注入！
    writer({"type": "progress", "step": "fetching"})
    return "result"

# 接收
for event in agent.stream(inputs, stream_mode="custom"):
    print(event)
```

**注意**：`StreamWriter` 通过 `get_stream_writer()` 函数获取，不是参数注入。

## 5. stream_mode="checkpoints"

返回检查点信息，用于状态持久化。需要配置 `checkpointer`。

```python
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model=model,
    checkpointer=InMemorySaver(),     # 必须配置
)

config = {"configurable": {"thread_id": "conv-001"}}  # 必须是 dict！

for checkpoint in agent.stream(inputs, stream_mode="checkpoints", config=config):
    cp_id = checkpoint["config"]["configurable"]["checkpoint_id"]
    step = checkpoint["metadata"]["step"]
    messages = checkpoint["checkpoint"]["channel_values"]["messages"]
```

**常见错误**：
```python
# ❌ 错误：用了逗号，变成 set
config = {"configurable": {"thread_id", "conv-001"}}

# ✅ 正确：用冒号，是 dict
config = {"configurable": {"thread_id": "conv-001"}}
```

## 6. stream_mode="tasks"

返回任务（节点执行）的开始和结束事件。

```python
for event in agent.stream(inputs, stream_mode="tasks"):
    data = event["data"]
    
    if "input" in data:
        # 开始事件
        print(f"[开始] {data['name']}")
    elif "result" in data:
        # 结束事件
        print(f"[结束] {data['name']}")
        if data["error"]:
            print(f"错误: {data['error']}")
```

**结构**：
- 每个任务返回 2 个事件（开始 + 结束）
- 开始事件包含 `input`, `triggers`
- 结束事件包含 `result`, `error`
- 通过 `data` 中的字段判断是开始还是结束（`type` 都是 `"tasks"`）

## 7. stream_mode="debug"

最详细的模式，包含所有调试信息。

```python
for event in agent.stream(inputs, stream_mode="debug"):
    if event["type"] == "task":
        print(f"[开始] 节点: {event['payload']['name']}")
        print(f"  输入: {event['payload']['input']}")
    elif event["type"] == "task_result":
        print(f"[完成] 节点: {event['payload']['name']}")
        print(f"  输出: {event['payload']['result']}")
```

## 模式对比

### values vs updates vs messages

```
         model 节点执行（生成 100 个 token）
         ─────────────────────────────────

values:  └── 返回完整 state（包含 HumanMessage）─────→
         1 次

updates: └── 返回增量（不包含 HumanMessage）─────────→
         1 次

messages: ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓
         每个 token 一次，共 100 次
```

### tasks vs messages

| 维度 | tasks | messages |
|------|-------|----------|
| 粒度 | 节点级 | token 级 |
| 频率 | 每个节点 2 次 | 每个 token 1 次 |
| 包含输入 | ✅ | ❌ |
| 包含错误 | ✅ | ❌ |

## stream() vs stream_events()

| 方法                | 来源                    | 说明                    |
| ----------------- | --------------------- | --------------------- |
| `stream()`        | LangGraph 特有          | 通过 `stream_mode` 选择模式 |
| `stream_events()` | LangChain Runnable 基类 | 返回所有生命周期事件            |

**注意**：`stream_mode="events"` **不存在**！events 只能通过 `stream_events()` 方法访问。

```python
# ✅ 正确
for event in agent.stream_events(inputs, version="v2"):
    print(event)

# ❌ 错误：stream_mode 没有 "events"
for event in agent.stream(inputs, stream_mode="events"):  # 报错！
    print(event)
```

## stream vs astream

| 方法 | 类型 | 说明 |
|------|------|------|
| `stream()` | 同步 | 阻塞当前线程 |
| `astream()` | 异步 | 不阻塞，可并发 |

```python
# 同步
for chunk in agent.stream(inputs, stream_mode="messages"):
    process(chunk)

# 异步
async for chunk in agent.astream(inputs, stream_mode="messages"):
    process(chunk)
```

## streaming 参数（模型级别）

```python
from langchain_openai import ChatOpenAI

# 模型级别的流式控制
model = ChatOpenAI(model="gpt-4o", streaming=False)  # 关闭流式
```

| 设置 | 行为 |
|------|------|
| `streaming=True`（默认） | 逐 token 流式返回 |
| `streaming=False` | 等全部生成完再一次性返回 |

**影响**：
- `stream_mode="messages"` 的逐 token 效果会消失
- `stream_mode="values"` / `"updates"` 仍然有效（节点级）

## Thinking / Reasoning 参数

| 模型 | 参数 | 说明 |
|------|------|------|
| Anthropic Claude | `thinking={"type": "enabled", "budget_tokens": 5000}` | 可精确控制 |
| OpenAI | `reasoning_effort="high"` | 只能选档位 |
| DeepSeek | `thinking=True` | 开/关 |

**thinking = reasoning**，只是不同厂商的叫法不同。

LangChain 统一了格式：所有 reasoning 内容都通过 `content_blocks` 的 `type="reasoning"` 访问。

## 多 Agent 系统

### 区分消息来源

```python
# 给每个 Agent 起名
main_agent = create_agent(model=model, name="main_agent", ...)
sub_agent = create_agent(model=model, name="weather_agent", ...)

# 流式时加 subgraphs=True
for chunk in agent.stream(inputs, stream_mode="messages", subgraphs=True):
    msg, metadata = chunk["data"]
    who = metadata.get("lc_agent_name", "unknown")
    print(f"[{who}] {msg.content}")
```

### subgraphs=True 的作用

- `subgraphs=False`（默认）：只看主 Agent，子 Agent 是黑盒
- `subgraphs=True`：穿透子 Agent，看到所有内部细节

## Human-in-the-Loop

```python
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.types import Command

agent = create_agent(
    model=model,
    tools=[dangerous_tool],
    middleware=[HumanInTheLoopMiddleware(interrupt_on={"get_weather": True})],
    checkpointer=InMemorySaver(),
)

# 第 1 轮：检测中断
interrupts = []
for chunk in agent.stream(inputs, stream_mode=["messages", "updates"], config=config):
    if chunk["type"] == "updates":
        if "__interrupt__" in chunk["data"]:
            interrupts.extend(chunk["data"]["__interrupt__"])

# 第 2 轮：用户审批后恢复
if interrupts:
    agent.invoke(Command(resume="approve"), config=config)
```

**interrupt 为什么出现在 updates 中**：`__interrupt__` 是 state 的特殊字段，updates 返回所有 state 变化。

## Accessing Completed Messages

在某些场景（如 guardrail middleware），完整消息不会自动出现在 `updates` 中：

```python
@after_agent
def safety_guardrail(state: AgentState, runtime: Runtime):
    # 获取 completed message
    last_message = state["messages"][-1]
    
    # 检查结果（对 updates 不可见）
    result = safety_model.invoke([...])
    
    # 用 stream_writer 手动发送
    writer = get_stream_writer()
    writer(result)
```

## 选择建议

| 场景 | 推荐模式 |
|------|---------|
| 聊天 UI 逐字显示 | `messages` |
| 调试/看状态变化 | `values` |
| 追踪哪个节点改了什么 | `updates` |
| 监控节点执行时间 | `tasks` |
| 深度调试 | `debug` |
| 自定义业务事件 | `custom` |
| 状态持久化/恢复 | `checkpoints` |
| 多模式组合 | `stream_mode=["messages", "updates"]` |

## 常见陷阱

1. **config 参数**：必须是 dict，不能用 set
   ```python
   # ❌ {"thread_id", "conv-001"}  这是 set
   # ✅ {"thread_id": "conv-001"}  这是 dict
   ```

2. **stream_mode="events" 不存在**：要用 `stream_events()` 方法

3. **tasks 事件判断**：`type` 都是 `"tasks"`，通过 `data` 中的字段区分开始/结束

4. **reasoning 模型的空 chunk**：`gpt-5-nano` 等会产生大量 `content: []`，是 thinking tokens

5. **Tool calling 限制**：reasoning 模型在 thinking 模式下，`tool_choice` 不能设为 `required`

6. **StreamWriter 获取**：通过 `get_stream_writer()` 函数，不是参数注入
