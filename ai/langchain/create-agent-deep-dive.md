# LangChain create_agent 详解

## 1. 函数签名

```python
from langchain.agents import create_agent

agent = create_agent(
    model: str | BaseChatModel,
    tools: Sequence[BaseTool | Callable | dict] | None = None,
    *,
    system_prompt: str | SystemMessage | None = None,
    middleware: Sequence[AgentMiddleware] = (),
    response_format: ResponseFormat | type | dict | None = None,
    state_schema: type[AgentState] | None = None,
    context_schema: type[ContextT] | None = None,
    checkpointer: Checkpointer | None = None,
    store: BaseStore | None = None,
    interrupt_before: list[str] | None = None,
    interrupt_after: list[str] | None = None,
    debug: bool = False,
    name: str | None = None,
    cache: BaseCache | None = None,
    transformers: Sequence[TransformerFactory] | None = None,
)
```

---

## 2. 核心参数详解

### 2.1 model（必填）

指定 Agent 使用的 LLM，支持两种格式：

**格式 A：字符串标识符**

```
"provider:model_name"
```

常见 provider 前缀：

| 前缀 | 提供商 | 所需包 |
|---|---|---|
| `openai:` | OpenAI | langchain-openai |
| `anthropic:` | Anthropic | langchain-anthropic |
| `deepseek:` | DeepSeek | langchain-deepseek |
| `google_vertexai:` | Google | langchain-google-vertexai |
| `mistralai:` | Mistral | langchain-mistralai |

```python
# 推荐：显式指定 provider
create_agent(model="openai:gpt-4o", tools=[search])
create_agent(model="anthropic:claude-sonnet-4-5-20250929", tools=[search])
create_agent(model="deepseek:deepseek-chat", tools=[search])
```

**格式 B：模型实例**

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o", temperature=0.7)
create_agent(model=model, tools=[search])
```

---

### 2.2 tools

Agent 可调用的工具列表，支持三种元素类型：

| 类型 | 说明 |
|---|---|
| `@tool` 装饰的函数 | 推荐，可自定义工具名称和参数描述 |
| 普通函数 | 靠 docstring 生成描述，复杂参数推断不准 |
| `dict` | 内置工具声明，如 `{"type": "tavily_search"}` |

**`@tool` 装饰器 vs 普通函数**

```python
# 普通函数：简单但元信息有限
def calculate(expression: str) -> str:
    """计算数学表达式"""
    return str(eval(expression))

# @tool 装饰器：可自定义名称、参数描述
from pydantic import BaseModel, Field

class CalcInput(BaseModel):
    expression: str = Field(description="数学表达式，如 '2+3*4'")
    precision: int = Field(default=2, description="小数精度")

@tool(args_schema=CalcInput)
def calculate(expression: str, precision: int = 2) -> str:
    """计算数学表达式"""
    result = eval(expression)
    return f"{result:.{precision}f}"
```

---

### 2.3 system_prompt

系统提示词，放在消息数组最前面：

```python
# 字符串
create_agent(model="openai:gpt-4o", system_prompt="你是专业助手。")

# SystemMessage 对象
from langchain_core.messages import SystemMessage
create_agent(
    model="openai:gpt-4o",
    system_prompt=SystemMessage(content="你是翻译助手。")
)
```

> 💡 需要动态提示词时用 middleware，不用这个参数。

---

### 2.4 checkpointer

持久化 Agent 状态，实现对话记忆：

```python
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="openai:gpt-4o",
    checkpointer=InMemorySaver()  # 内存存储，进程退出就丢
)

# 用 thread_id 标识对话
config = {"configurable": {"thread_id": "conv-001"}}
agent.invoke({"messages": [("user", "我叫张三")]}, config)
agent.invoke({"messages": [("user", "我叫什么")]}, config)  # 记住了
```

**thread_id 要点**：
- 固定 key，不能改名（不能写成 session_id）
- 值由你自己定，才能和业务系统对应
- 不传 checkpointer 就不需要 thread_id

**InMemorySaver**：
- 存在 Python 进程内存里（一个 dict）
- 进程退出数据丢失
- 适合开发测试，生产用 `PostgresSaver`、`SqliteSaver`

---

### 2.5 context_schema

定义不可变的运行时上下文，和 state_schema 的区别：

| 属性 | state_schema | context_schema |
|---|---|---|
| 可变性 | 可变，执行中会更新 | 不可变，调用时传入 |
| 生命周期 | 整个对话 | 单次调用 |
| 典型用途 | 消息历史、中间结果 | 用户信息、权限 |

```python
from dataclasses import dataclass

@dataclass
class Context:
    user_id: str

agent = create_agent(
    model="openai:gpt-4o",
    context_schema=Context,
)

# 调用时传入 context
agent.invoke(
    {"messages": [("user", "你好")]},
    config={"configurable": {"thread_id": "conv-001"}},
    context=Context(user_id="user-123"),
)
```

> ⚠️ 用 `@dataclass` 定义 Context，如果 invoke 的 context 参数有 Pydantic 序列化警告，把 context 放到 config 里：
> ```python
> config={"configurable": {"thread_id": "...", "context": Context(user_id="...")}}
> ```

---

### 2.6 state_schema

自定义 Agent 状态，添加额外字段：

```python
from typing import Annotated
from langchain.agents import AgentState

class MyState(AgentState):
    user_name: str = ""                                    # 覆盖模式
    visited_urls: Annotated[list[str], list.__add__]       # 追加模式

# reducer 决定合并方式：
# - 不加 Annotated → 覆盖
# - Annotated[type, list.__add__] → 追加
# - Annotated[type, max] → 取最大值
# - Annotated[type, 自定义函数] → 自定义合并
```

**工具如何更新状态字段**

工具通过返回 `Command` 对象显式更新，Agent（LLM）不直接操作这些字段：

```python
from langgraph.types import Command, ToolMessage
from langchain.tools import tool

@tool
def search(query: str) -> Command:
    """搜索信息"""
    return Command(
        update={
            "messages": [ToolMessage(content="结果...", tool_call_id=...)],
            "visited_urls": ["https://example.com"],  # 键名必须匹配 state_schema 字段
        }
    )
```

---

### 2.7 interrupt_before / interrupt_after

在指定节点前/后暂停，实现人工审批：

```python
agent = create_agent(
    model="openai:gpt-4o",
    tools=[dangerous_tool],
    interrupt_before=["tools"],  # 工具执行前暂停
    checkpointer=InMemorySaver(),
)

# 第一次调用，Agent 暂停
result = agent.invoke({"messages": [...]}, config)

# 检查中断
if result.get("__interrupt__"):
    # 前端展示审批界面

# 用户批准后恢复
from langgraph.types import Command
result = agent.invoke(Command(resume="approve"), config)
```

---

## 3. 工具函数参数

工具函数可以接收两类参数：

### 3.1 用户定义参数（LLM 填）

```python
@tool
def search(query: str, max_results: int = 5) -> str:
    """搜索信息"""
    ...
```

### 3.2 框架注入参数（通过 ToolRuntime）

```python
from langchain.tools import tool, ToolRuntime

@tool
def search(query: str, runtime: ToolRuntime) -> str:
    """搜索信息"""
    # runtime 框架自动注入，对 LLM 隐藏
    state = runtime.state              # 当前 Agent 状态
    context = runtime.context          # 不可变上下文
    store = runtime.store              # 长期存储
    config = runtime.config            # RunnableConfig
    tool_call_id = runtime.tool_call_id
    stream_writer = runtime.stream_writer
    execution_info = runtime.execution_info
    server_info = runtime.server_info
    ...
```

**保留名称**：`config` 和 `runtime` 不能用作普通输入参数。

---

## 4. Middleware

Middleware 在 Agent 执行的各阶段插入逻辑。

### 4.1 拦截点

```
用户输入 → [before_model] → LLM → [after_model]
                                        ↓
                                  [before_tool] → 工具 → [after_tool]
```

### 4.2 常见 middleware

| Middleware | 用途 |
|---|---|
| `FilesystemMiddleware` | 文件读写，backend 指定存储位置 |
| `SummarizationMiddleware` | 对话太长时自动压缩摘要 |
| `MemoryMiddleware` | 加载知识库（如 AGENTS.md）供 Agent 检索 |
| `SkillsMiddleware` | 加载技能定义（如 ./skills/ 目录） |
| `ProviderToolSearchMiddleware` | 动态搜索可用工具 |
| `FaultToleranceMiddleware` | 自动处理限流、超时、重试 |
| `HumanInTheLoopMiddleware` | 人工审批拦截 |

### 4.3 FilesystemMiddleware 的 backend

```python
from langchain.agents.middleware import FilesystemMiddleware
from langchain.agents.middleware.filesystem import StateBackend

# StateBackend：文件存在 Agent 状态（内存）里，不碰磁盘
backend = StateBackend()
FilesystemMiddleware(backend=backend)
```

### 4.4 Guardrails（安全拦截）

Guardrails 是基于 middleware 实现的安全检查，通常在 `after_tool` 拦截：

```python
class ContentGuardrail(AgentMiddleware):
    def after_tool(self, tool_output, ...):
        if contains_sensitive_info(tool_output):
            return redact(tool_output)  # 脱敏
        return tool_output
```

---

## 5. 完整示例

```python
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI
from langchain.tools import tool, ToolRuntime
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command
import uuid7

@dataclass
class Context:
    user_id: str

@tool
def get_weather(city: str, runtime: ToolRuntime) -> str:
    """获取天气"""
    user_id = runtime.context.user_id
    print(f"用户 {user_id} 查询 {city} 天气")
    return f"{city}：晴天，25°C"

model = ChatOpenAI(model="deepseek:deepseek-chat")

agent = create_agent(
    model=model,
    tools=[get_weather],
    context_schema=Context,
    checkpointer=InMemorySaver(),
)

# 第一次调用
config = {"configurable": {"thread_id": str(uuid7())}}
result = agent.invoke(
    {"messages": [{"role": "user", "content": "San Francisco 天气怎么样?"}]},
    config=config,
    context=Context(user_id="user-123"),
)
print(result["messages"][-1].content)

# 第二次调用，同一个 thread_id，对话继续
result = agent.invoke(
    {"messages": [{"role": "user", "content": "明天呢?"}]},
    config=config,
)
print(result["messages"][-1].content)
```

---

## 6. 关键概念

### Agent = LLM + Harness

- LLM 负责思考
- Harness 是包裹 LLM 的软件框架，管理工具、记忆、上下文、流程控制

### Delegation（委派）

复杂任务拆给多个子 Agent 并行处理，每个子 Agent 有独立上下文：

```
主 Agent（协调）
    ├── 子 Agent A（独立上下文）
    ├── 子 Agent B（独立上下文）
    └── 子 Agent C（独立上下文）
```

### Steering（引导/人工介入）

高风险操作暂停等人工审批，不改动 Agent 结构：

- 批准 → 继续执行
- 修改 → 改参数后继续
- 拒绝 → 取消操作

### Fault Tolerance（容错）

容错 middleware 统一处理限流、超时、临时错误，工具代码不用写 try/except。
