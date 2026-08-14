# OpenAI Agents SDK - Running Agents

## 一、核心概念：Agent Loop

Agent 的运行是一个循环：

```
用户输入 → Agent 思考 → 有工具调用？ → 执行 → 喂回结果 → 再思考
                      ↓ 没有
                   返回最终回答
```

- 一次 `Runner.run()` = 一个应用层回合（turn）
- 内部可能跑多步（工具调用、handoff）
- `max_turns` 控制循环步数，防止死循环

---

## 二、四种对话状态策略

| 策略 | 状态在哪 | 传什么 | 适用场景 |
|------|---------|--------|---------|
| `result.to_input_list()` | 你的应用 | 完整历史 | 开发调试、要控制历史 |
| `session` | 你的存储 + SDK | 同一个 session 对象 | 生产应用、要持久化 |
| `conversationId` | OpenAI 服务端 | 同一个 conversation ID | 多 worker、微服务 |
| `previous_response_id` | OpenAI 服务端 | 上一个 response ID | 简单链式对话 |

**关键区别**：
- 越往左（本地），控制越多，责任越重
- 越往右（服务端），越轻松，控制越少

### 案例 1：to_input_list()

```python
import asyncio
from agents import Agent, Runner

async def main():
    agent = Agent(
        name="聊天助手",
        instructions="你是一个友好的助手，会记住用户说过的信息。用中文回答。",
    )

    history = None

    while True:
        user_input = input("你:").strip()
        if user_input.lower() == "exit":
            print("再见")
            break
        if not user_input:
            continue

        # 第一轮传字符串，后续传历史
        if not history:
            result = await Runner.run(agent, user_input)
        else:
            history.append({"role": "user", "content": user_input})
            result = await Runner.run(agent, input=history)

        print(f"助手: {result.final_output}\n")

        # 用覆盖，不用追加
        history = result.to_input_list()
        print(f"[调试] 当前历史条数: {len(history)}\n")

if __name__ == "__main__":
    asyncio.run(main())
```

### 案例 2：session（SQLite 持久化）

```python
import asyncio
from agents import Agent, Runner, SQLiteSession

async def main():
    agent = Agent(
        name="聊天助手",
        instructions="你是一个友好的助手，会记住用户说过的信息。用中文回答。",
    )

    session_id = "user_alice_session"
    session = SQLiteSession(session_id, "chat_sessions.db")

    while True:
        user_input = input("你:").strip()
        if user_input.lower() == "quit":
            print("再见！下次可以继续聊。")
            break
        if not user_input:
            continue

        result = await Runner.run(
            agent,
            user_input,
            session=session,  # 传同一个 session 对象
        )

        print(f"助手: {result.final_output}\n")

if __name__ == "__main__":
    asyncio.run(main())
```

**特点**：
- 历史存在 SQLite 数据库里
- 程序重启后，用同一个 `session_id` 还能继续聊
- SDK 自动管理历史，不用手动 `to_input_list()`

### 案例 3：conversationId（OpenAI 服务端管理）

```python
import asyncio
from agents import Agent, Runner

async def main():
    agent = Agent(
        name="聊天助手",
        instructions="你是一个友好的助手，会记住用户说过的信息。用中文回答。",
    )

    conversation_id = "conv_demo_001"

    while True:
        user_input = input("你:").strip()
        if user_input.lower() == "quit":
            print("再见！")
            break
        if not user_input:
            continue

        result = await Runner.run(
            agent,
            user_input,
            conversation_id=conversation_id,  # 同一个 ID，服务端自动关联历史
        )

        print(f"助手: {result.final_output}\n")

if __name__ == "__main__":
    asyncio.run(main())
```

**特点**：
- 状态存在 OpenAI 服务端
- 多个 worker 可以用同一个 `conversation_id` 访问对话
- 每轮只传新内容，不用传完整历史

### 案例 4：previous_response_id（最轻量）

```python
import asyncio
from agents import Agent, Runner

async def main():
    agent = Agent(
        name="聊天助手",
        instructions="你是一个友好的助手。用中文回答。",
    )

    first_input = input("你:").strip()
    if first_input.lower() == "quit":
        return

    result = await Runner.run(agent, first_input)
    print(f"助手: {result.final_output}")

    response_id = result.response_id

    while True:
        user_input = input("你:").strip()
        if user_input.lower() == "quit":
            print("再见！")
            break
        if not user_input:
            continue

        result = await Runner.run(
            agent,
            user_input,
            previous_response_id=response_id,  # 从上一次响应继续
        )

        print(f"助手: {result.final_output}")
        response_id = result.response_id

if __name__ == "__main__":
    asyncio.run(main())
```

**特点**：
- 最轻量，只传一个 ID
- 只能线性延续，不能分支
- 历史对你是黑盒，存在 OpenAI 服务端

---

## 三、代码踩坑记录

### 错误 1：参数名错

```python
# ❌ 错误
result = await Runner.run(starting_agent=agent, input=history)

# ✅ 正确
result = await Runner.run(agent, input=history)
```

### 错误 2：[] is None 为 False

```python
history = []

# ❌ 错误
if history is None:  # False！空列表不是 None

# ✅ 正确
if not history:  # True，空列表是 falsy
```

### 错误 3：历史重复

```python
# ❌ 错误：用户消息出现两次
history.append({"role":"user","content":user_input})
result = await Runner.run(agent, input=history)
history += result.to_input_list()  # to_input_list() 也包含这条消息！

# ✅ 正确：用覆盖，不用追加
history.append({"role":"user","content":user_input})
result = await Runner.run(agent, input=history)
history = result.to_input_list()
```

---

## 四、流式输出（Streaming）

```python
async with Runner.stream(agent, "写一首诗") as stream:
    async for event in stream.stream_events():
        if event.type == "raw_model_stream_event":
            print(event.data.delta, end="")
    
    result = await stream.final_output()  # 流结束后拿完整结果
```

### 三条实用规则

1. **等流跑完再视为结束**：不要提前退出 `stream_events()` 循环
2. **审批暂停后，从状态恢复**：不开新回合
3. **取消流后，用 session/ID 从状态恢复**：不要从头开始

---

## 五、审批流程（Approvals）

```python
from agents import Agent, Runner, function_tool

@function_tool
def delete_file(path: str):
    """删除文件"""
    import os
    os.remove(path)
    return f"已删除 {path}"

agent = Agent(
    name="文件助手",
    tools=[delete_file],
)

async def main():
    result = await Runner.run(
        agent,
        "删除 test.txt",
        tool_use_behavior={
            "delete_file": "require_approval"  # 需要人工审批
        }
    )

    if result.has_pending_approvals():
        print("=== 等待审批 ===")
        
        for approval in result.pending_approvals:
            print(f"工具: {approval.tool_name}")
            print(f"参数: {approval.arguments}")
            
            user_confirm = input("批准？(y/n): ")
            if user_confirm == "y":
                approval.approve()
            else:
                approval.reject("用户拒绝")
        
        # 从暂停状态恢复
        result = await Runner.run(
            agent,
            input=result.to_input_list(),
        )
        
        print(f"完成: {result.final_output}")

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

---

## 六、暂停 vs 失败

| 类型 | 含义 | 处理方式 |
|------|------|---------|
| **失败** | max_turns 超限、工具报错 | 捕获异常 |
| **暂停** | 人工审批 | 从状态恢复 |

**关键**：把审批当暂停，不当新回合，否则轮数、历史、ID 会乱。

---

## 七、Runner.run() 完整签名

```python
await Runner.run(
    agent,                    # 第1个位置参数：Agent 实例
    input,                    # 第2个位置参数：输入（字符串或消息列表）
    *,                        # 后面都是关键字参数
    session=None,             # Session 对象
    conversation_id=None,     # OpenAI Conversation ID
    previous_response_id=None,# 上一个 response ID
    max_turns=10,             # 最大工具调用轮数
    run_config=None,          # 运行配置
    tool_use_behavior=None,   # 工具审批策略
)
```

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `agent` | `Agent` | 要运行的 Agent 实例 |
| `input` | `str \| list` | 用户输入（字符串）或历史消息列表 |
| `session` | `Session` | Session 对象，用于持久化 |
| `conversation_id` | `str` | OpenAI Conversation ID |
| `previous_response_id` | `str` | 上一个 response ID |
| `max_turns` | `int` | 最大循环轮数，默认 10 |
| `run_config` | `RunConfig` | 模型配置覆盖等 |
| `tool_use_behavior` | `dict` | 工具审批策略 |

---

## 八、待学内容

- [ ] 工具系统详解（`@function_tool`、内置工具）
- [ ] Handoff 多 Agent 协作
- [ ] Guardrails 护栏
