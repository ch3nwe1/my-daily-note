# LangChain Agent 短期记忆与上下文管理

> 学习日期：2026-08-21

---

## 一、短期记忆的核心概念

### 1.1 什么是短期记忆

短期记忆（Short-term memory）是让 Agent 在单个会话线程中记住之前交互的能力。实现方式：

- **对话历史**：最常见的形式，记录所有消息
- **线程级持久化**：通过 checkpointer 按 `thread_id` 隔离不同会话
- **状态管理**：作为 graph state 的一部分，自动更新和恢复

### 1.2 上下文窗口的挑战

长对话对 LLM 构成三大挑战：

| 问题 | 表现 |
|------|------|
| **上下文超限** | 超过模型硬限制，API 直接报错 |
| **性能下降** | 响应变慢、成本飙升、注意力稀释 |
| **迷失在中间** | 模型忽略中间内容，只记住开头和结尾 |

### 1.3 上下文的完整构成

```
发送给模型的完整上下文 = 
  系统提示词（10-20%）
  + 对话历史（50-70%，主要部分）
  + 工具调用结果（15-30%）
  + 注入的额外信息（RAG 等，0-10%）
```

---

## 二、上下文裁剪策略

### 2.1 五种裁剪方式

| 策略             | 原理                  | 适用场景     |
| -------------- | ------------------- | -------- |
| **按数量裁剪**      | 保留最近 N 条消息          | 简单聊天     |
| **按 token 裁剪** | 保留最近 N 个 token      | 消息长短不一   |
| **语义摘要**       | 旧消息压缩成摘要            | 长对话（推荐）  |
| **保留系统消息**     | system message 永远保留 | 所有场景     |
| **按重要性过滤**     | 基于相关性打分             | 复杂 Agent |

### 2.2 语义摘要的实现

#### ConversationSummaryBufferMemory 原理

```python
class ConversationSummaryBufferMemory:
    buffer: List[BaseMessage]           # 最近的原始消息
    moving_summary_buffer: str = ""     # 累积的摘要
    
    def save_context(self, input, output):
        # 1. 加入新消息
        self.buffer.extend([HumanMessage(input), AIMessage(output)])
        
        # 2. 超阈值就压缩
        while self._get_num_tokens(self.buffer) > self.max_token_limit:
            # 取前半部分
            old_msgs = self.buffer[:len(self.buffer)//2]
            
            # 增量摘要：旧摘要 + 旧消息 → 新摘要
            self.moving_summary_buffer = self.predict_new_summary(
                messages=old_msgs,
                existing_summary=self.moving_summary_buffer
            )
            
            # 删除已摘要的消息
            self.buffer = self.buffer[len(self.buffer)//2:]
    
    def load_memory_variables(self):
        if self.moving_summary_buffer:
            # 摘要作为 SystemMessage 放在最前
            return [SystemMessage(content=self.moving_summary_buffer)] + self.buffer
        return self.buffer
```

**关键机制**：
- 按 token 阈值触发（不是消息条数）
- 增量摘要（旧摘要 + 新消息 → 新摘要）
- 摘要作为 SystemMessage 放在消息列表开头
- buffer 只删前半，保留后半（最新的）

#### 局限性

| 问题 | 说明 |
|------|------|
| 摘要是字符串，非结构化 | 不能单独查询特定信息 |
| 滚动摘要会累积误差 | 摘要上再生成摘要，细节丢失 |
| 摘要和原始消息混在一起 | 无法跨多个对话共享 |
| 没有持久化 | 进程结束 memory 就没了 |

---

## 三、完整历史保存 vs 上下文裁剪

### 3.1 核心区分

```
┌─────────────────────────────────────────────┐
│  上下文（Context）                           │
│  送给 LLM 的内容                             │
│  必须精简，受 token 限制                     │
│  → 可以摘要、裁剪                           │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│  持久化存储（Storage）                       │
│  完整的对话历史                              │
│  没有长度限制                                │
│  → 必须完整保存，不丢一条                    │
└─────────────────────────────────────────────┘
```

**两者独立**：可以一边压缩上下文，一边把完整对话存到别的地方。

### 3.2 三种保存方案

#### 方案一：数据库自存

```python
import sqlite3

# 完整记录每条消息
db.execute("""
    CREATE TABLE messages (
        session_id TEXT,
        role TEXT,
        content TEXT,
        timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
    )
""")

def save_message(session_id, role, content):
    db.execute(
        "INSERT INTO messages (session_id, role, content) VALUES (?, ?, ?)",
        (session_id, role, content)
    )
    db.commit()
```

#### 方案二：LangGraph Checkpointer（推荐）

```python
from langgraph.checkpoint.sqlite import SqliteSaver

checkpointer = SqliteSaver.from_conn_string("chat.db")
app = graph.compile(checkpointer=checkpointer)

# 每次调用指定 thread_id
config = {"configurable": {"thread_id": "session_123"}}
app.invoke({"messages": [HumanMessage("你好")]}, config)

# 完整历史自动保存，包括每一步的中间状态
state = app.get_state(config)
full_messages = state.values["messages"]  # 一条都没丢
```

#### 方案三：两者结合（生产环境）

```
存储层（Checkpointer / DB）
  ── 完整保存所有消息，永久不丢
  ── 可以按时间、关键词、session 查询
  ── 可以"时间旅行"回到任意历史时刻
            ↑
            │ 读取完整历史
上下文构造层（自定义逻辑）
  ── 从完整历史里挑选：
      · 最近 N 条原始消息
      · 旧消息的摘要
      · 与当前问题相关的关键消息
  ── 拼装成送给 LLM 的消息列表
```

---

## 四、Checkpointer 与 Agent 集成

### 4.1 创建带 checkpointer 的 Agent

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model=model,
    tools=[get_user_info],
    checkpointer=InMemorySaver(),  # 启用短期记忆
)

# 使用 thread_id 隔离不同会话
thread_config = {"configurable": {"thread_id": "1"}}

# 第一次调用
response = agent.invoke(
    {"messages": [HumanMessage("Hi! My name is Bob.")]},
    config=thread_config,
)

# 第二次调用（能记住第一次的内容）
response = agent.invoke(
    {"messages": [HumanMessage("What's my name?")]},
    config=thread_config,
)
# 模型会回答："Your name is Bob."
```

### 4.2 调用格式

| 组件 | invoke 输入格式 | 原因 |
|------|----------------|------|
| **ChatOpenAI** | `model.invoke([HumanMessage(...)])` | 单步调用，接收消息序列 |
| **Agent (CompiledGraph)** | `agent.invoke({"messages": [...]})` | 整图执行，接收 state 结构 |

**简化写法**：

```python
# 辅助函数
def chat(text: str, config: dict) -> str:
    response = agent.invoke(
        {"messages": [HumanMessage(text)]},
        config=config,
    )
    return response["messages"][-1].content

# 使用
print(chat("你好", thread_config))
```

---

## 五、中间件机制：@before_model 与 @after_model

### 5.1 执行时机

```
Agent 循环：

while True:
    @before_model()           # 每轮都执行
    ai_message = call_llm()   # 调用模型
    @after_model()            # 每轮都执行
    
    if ai_message.has_tool_calls():
        execute_tools()       # 执行工具
        # 循环继续，再次调用模型
    else:
        return ai_message     # 没有工具调用，结束
```

**关键**：不管模型是决定调用工具还是给出最终回复，两个装饰器都会执行。

### 5.2 典型用途

| 装饰器 | 时机 | 用途 |
|--------|------|------|
| **@before_model** | 模型调用前 | 裁剪历史、注入提示、权限检查 |
| **@after_model** | 模型返回后 | 过滤输出、记录日志、敏感信息检查 |

### 5.3 实现消息裁剪

```python
from langchain_core.messages import trim_messages, RemoveMessage, REMOVE_ALL_MESSAGES

@before_model
def trim_messages_middleware(state: AgentState, runtime: Runtime):
    """Keep only the last few messages to fit context window."""
    messages = state["messages"]
    
    if len(messages) <= 3:
        return None  # No changes needed
    
    first_msg = messages[0]  # 保留系统消息
    recent_messages = messages[-3:] if len(messages) % 2 == 0 else messages[-4:]
    new_messages = [first_msg] + recent_messages
    
    return {
        "messages": [
            RemoveMessage(id=REMOVE_ALL_MESSAGES),  # 删除所有
            *new_messages                            # 添加裁剪后的
        ]
    }

# 挂载到 agent
agent = create_agent(
    model=model,
    tools=[...],
    middleware=[trim_messages_middleware],
    checkpointer=InMemorySaver(),
)
```

### 5.4 返回值含义

```python
# 返回 dict：更新 state
return {"messages": [...]}

# 返回 None：不修改 state
return None
```

**状态更新语法**：

```python
{
    "messages": [
        RemoveMessage(id=REMOVE_ALL_MESSAGES),  # 操作 1：删除所有
        *new_messages                            # 操作 2：添加新消息
    ]
}
```

**为什么这样写**：`messages` 字段的 reducer 是 `add_messages`（追加模式），直接赋值会被当作追加。必须先删后加才能实现替换。

---

## 六、消息历史的合法性约束

### 6.1 两条重要限制

| 限制 | 原因 | 违反后果 |
|------|------|---------|
| **必须以用户消息开头** | 某些模型的硬性要求 | API 报错 |
| **工具调用必须有对应结果** | 模型推理依赖工具返回值 | 模型困惑，回答错乱 |

### 6.2 示例

```
❌ 非法（以 assistant 开头）：
  [AI]:    你好！
  [Human]: 你好

✅ 合法：
  [Human]: 你好
  [AI]:    你好！

❌ 非法（工具调用后没有结果）：
  [Human]: 今天天气
  [AI]:    （调用 get_weather）   ← 有工具调用
  [Human]: 明天呢                 ← 但没有工具结果！

✅ 合法：
  [Human]: 今天天气
  [AI]:    （调用 get_weather）
  [Tool]:  晴，25°C              ← 紧跟工具结果
  [AI]:    今天晴
```

### 6.3 安全的裁剪实现

```python
@before_model
def safe_trim_messages(state, runtime):
    messages = state["messages"]
    
    # 1. 保留系统消息
    system_msg = messages[0] if messages[0].type == "system" else None
    start_idx = 1 if system_msg else 0
    
    # 2. 从后往前找，确保不破坏工具调用配对
    recent = []
    i = len(messages) - 1
    while i >= start_idx and len(recent) < 4:
        msg = messages[i]
        
        # 如果是工具结果，必须连同前面的工具调用一起保留
        if msg.type == "tool":
            recent.append(msg)
            i -= 1
            if i >= start_idx and messages[i].type == "ai" and messages[i].tool_calls:
                recent.append(messages[i])
                i -= 1
        else:
            recent.append(msg)
            i -= 1
    
    recent.reverse()  # 恢复顺序
    
    # 3. 组装新消息
    new_messages = ([system_msg] if system_msg else []) + recent
    
    return {
        "messages": [
            RemoveMessage(id=REMOVE_ALL_MESSAGES),
            *new_messages
        ]
    }
```

---

## 七、Python 语法补充

### 7.1 TypedDict 的 total 参数

```python
class RunnableConfig(TypedDict, total=False):
    ...
```

| `total` 值 | 含义 |
|-----------|------|
| `total=True`（默认） | 所有 key 都是**必填**的 |
| `total=False` | 所有 key 都是**可选**的 |

**注意**：`TypedDict` 不是真正的继承，是 Python 的特殊语法。`total` 是传给类型系统的参数，运行时不起作用（只影响类型检查和 IDE 提示）。

### 7.2 tools=[...] 的含义

在文档代码中：

```python
agent = create_agent(
    model=model,
    tools=[...],  # ← 占位符，表示"这里填你的工具"
)
```

`[...]` 是文档占位符，不是空列表。真正的空列表是 `[]`。

在 Python 里，`[...]` 是包含一个 `Ellipsis` 对象的列表：

```python
empty = []
print(len(empty))        # 0

ellipsis_list = [...]
print(len(ellipsis_list))  # 1
print(ellipsis_list[0])    # Ellipsis
```

---

## 八、核心概念总结

### 8.1 短期记忆 = State + Checkpointer

```
State（容器）
  ── 存储当前线程的消息历史
  ── 作为 graph 的一部分自动更新
  
Checkpointer（仓库）
  ── 把 state 持久化到数据库
  ── 按 thread_id 隔离不同会话
  ── 支持时间旅行（回到任意历史时刻）
```

### 8.2 上下文管理 = 裁剪 + 存储（独立的两件事）

```
裁剪（为了 LLM）
  ── 按 token 限制精简消息
  ── 可以用摘要、裁剪、过滤等策略
  ── 通过 @before_model 中间件自动执行

存储（为了不丢东西）
  ── 完整保存所有消息
  ── 通过 Checkpointer 自动持久化
  ── 可以随时查询、导出、分析
```

### 8.3 中间件执行时机

```
每次模型调用：
  @before_model  →  调用 LLM  →  @after_model
       ↑                                ↓
       └────── 工具执行（如果有）←──────┘
              ↓
         回到 @before_model（循环）
```

**关键**：不管模型决定调用工具还是给出最终回复，装饰器都会执行。工具调用会触发下一轮循环。

---

## 九、实践建议

### 9.1 生产环境配置

```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_agent
from langgraph.checkpoint.postgres import PostgresSaver
from langchain_core.messages import trim_messages

# 1. 配置模型
model = ChatOpenAI(model="gpt-4o-mini")

# 2. 定义工具
def get_user_info() -> str:
    """查询用户信息"""
    return "用户名：小明，城市：北京"

# 3. 定义中间件
@before_model
def trim_history(state, runtime):
    messages = state["messages"]
    if len(messages) > 10:
        return {
            "messages": [
                RemoveMessage(id=REMOVE_ALL_MESSAGES),
                *messages[-10:]
            ]
        }
    return None

# 4. 创建 agent
agent = create_agent(
    model=model,
    tools=[get_user_info],
    middleware=[trim_history],
    checkpointer=PostgresSaver.from_conn_string("postgresql://..."),
)

# 5. 使用
config = {"configurable": {"thread_id": "session_123"}}
response = agent.invoke(
    {"messages": [HumanMessage("你好")]},
    config=config,
)
```

### 9.2 封装简化调用

```python
class ChatSession:
    def __init__(self, agent, thread_id: str):
        self.agent = agent
        self.config = {"configurable": {"thread_id": thread_id}}
    
    def say(self, text: str) -> str:
        response = self.agent.invoke(
            {"messages": [HumanMessage(text)]},
            config=self.config,
        )
        return response["messages"][-1].content

# 使用
session = ChatSession(agent, thread_id="1")
print(session.say("你好"))
print(session.say("我叫什么？"))
```

---

## 十、关键术语对照

| 英文 | 中文 |
|------|------|
| Short-term memory | 短期记忆 |
| Thread-level persistence | 线程级持久化 |
| Checkpointer | 检查点器 |
| Context window | 上下文窗口 |
| Conversation history | 对话历史 |
| Message trimming | 消息裁剪 |
| Semantic summarization | 语义摘要 |
| Middleware | 中间件 |
| State | 状态 |
| Tool call | 工具调用 |
| Tool result | 工具结果 |

---

## 十一、参考资源

- LangChain 官方文档：Memory 章节
- LangGraph Checkpointer 使用指南
- ConversationSummaryBufferMemory 源码
- @before_model / @after_model 中间件文档

---

**文档生成时间**：2026-08-21  
**会话主题**：LangChain Agent 短期记忆、上下文管理、中间件机制
