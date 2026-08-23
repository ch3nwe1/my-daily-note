# LangChain Middleware 中间件详解

## 1. 概述

Middleware（中间件）是一种在 Agent 执行流程的各个阶段插入自定义逻辑的机制。它不是独立的运行时，而是运行在 `create_agent` 返回的编译后的 LangGraph 内部。

### 适用场景

- 追踪 Agent 行为（日志、分析、调试）
- 转换 prompt、工具选择、输出格式
- 添加重试、回退、提前终止逻辑
- 施加速率限制、安全护栏、PII 检测
- 人工审批和介入

### 使用方式

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[SummarizationMiddleware(...)],
)
```

---

## 2. Hook 类型（拦截点）

Middleware 提供 6 种 hook，分为两类：

### 2.1 Node-Style Hooks（节点式）

在特定时机顺序执行，通过返回字典修改 Agent 状态。

| Hook | 执行时机 | 能力 |
|---|---|---|
| `before_agent` | Agent 启动前（每次调用只执行一次） | 修改 Agent 状态 |
| `before_model` | 每次模型调用前 | 修改状态 + 控制流程（可跳转到 end/tools/model） |
| `after_model` | 每次模型响应后 | 修改状态 + 控制流程 |
| `after_agent` | Agent 完成后（每次调用只执行一次） | 修改 Agent 状态 |

### 2.2 Wrap-Style Hooks（包裹式）

拦截执行过程，可以控制底层处理器执行零次、一次或多次。

| Hook | 执行时机 | 能力 |
|---|---|---|
| `wrap_model_call` / `awrap_model_call` | 包裹每次模型调用 | 可修改 ModelRequest（覆盖模型、工具、系统消息），可重试 |
| `wrap_tool_call` / `awrap_tool_call` | 包裹每次工具调用 | 拦截工具执行，可返回 Command 直接更新状态 |

### 2.3 完整生命周期

```
调用 Agent
  → [before_agent]          # 只执行一次
    → 循环开始
      → [before_model]      # 可跳转 end/tools/model
      → [wrap_model_call]   # 包裹模型调用（可重试/修改请求）
        → 实际模型调用
      → [after_model]       # 可跳转
      → [wrap_tool_call]    # 包裹工具调用
        → 实际工具执行
      → [after_tool]        # 工具执行后
    → 循环结束
  → [after_agent]           # 只执行一次
```

### 2.4 同步/异步分发

| 调用方式 | 触发的 hook |
|---|---|
| `agent.invoke(...)` | `wrap_model_call`, `wrap_tool_call` |
| `agent.ainvoke(...)` | `awrap_model_call`, `awrap_tool_call` |

**注意：** 只定义 sync 版本但用 ainvoke 会报 `NotImplementedError`。

---

## 3. 内置 Middleware

### 3.1 ToolErrorMiddleware — 工具错误处理

捕获工具执行异常，转换为错误的 ToolMessage 让模型能看到并恢复。

```python
from langchain.agents.middleware import ToolErrorMiddleware

def on_error(exc: Exception, request: ToolCallRequest) -> str | None:
    if isinstance(exc, ConnectionError):
        return f"Tool `{request.tool_call['name']}` encountered a connection error."
    return None  # 返回 None 表示异常继续传播

middleware = ToolErrorMiddleware(
    on_error=on_error,           # 同步错误回调
    aon_error=aon_error,         # 异步错误回调（可选）
    tools=["search_api"],        # 限定作用范围（可选，默认所有工具）
)
```

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `on_error` | `OnError \| None` | `None` | 同步错误处理回调 |
| `aon_error` | `AOnError \| None` | `None` | 异步错误处理回调 |
| `tools` | `list[BaseTool \| str] \| None` | `None` | 限定作用范围的工具列表 |

### 3.2 ToolRetryMiddleware — 工具重试

自动重试失败的工具调用，支持指数退避。

```python
from langchain.agents.middleware import ToolRetryMiddleware

middleware = ToolRetryMiddleware(
    max_retries=3,              # 最大重试次数
    retry_on=(ConnectionError, TimeoutError),  # 重试条件
    on_failure="continue",      # 重试耗尽后的行为
    backoff_factor=2.0,         # 指数退避乘数
    initial_delay=1.0,          # 初始延迟秒数
    max_delay=60.0,             # 最大延迟秒数
    jitter=True,                # 是否加 ±25% 随机抖动
    tools=["search_api"],       # 限定作用范围
)
```

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `max_retries` | `int` | `2` | 最大重试次数（≥0） |
| `tools` | `list[BaseTool \| str] \| None` | `None` | 限定作用范围 |
| `retry_on` | `tuple[type[Exception], ...] \| Callable` | `default_retry_on` | 重试条件 |
| `on_failure` | `'continue' \| 'error' \| Callable` | `'continue'` | 重试耗尽后行为 |
| `backoff_factor` | `float` | `2.0` | 指数退避乘数，0.0 表示固定延迟 |
| `initial_delay` | `float` | `1.0` | 初始延迟秒数 |
| `max_delay` | `float` | `60.0` | 最大延迟秒数 |
| `jitter` | `bool` | `True` | 是否加随机抖动避免惊群效应 |

**Jitter（抖动）解释：** 在重试延迟上添加 ±25% 的随机变化，防止多个请求同时重试导致服务器再次过载。

### 3.3 ModelRetryMiddleware — 模型重试

自动重试失败的模型调用，支持指数退避。参数与 ToolRetryMiddleware 类似。

### 3.4 ModelFallbackMiddleware — 模型回退

当主模型失败时，自动回退到备选模型。

```python
from langchain.agents.middleware import ModelFallbackMiddleware

# 按顺序尝试备选模型
middleware = ModelFallbackMiddleware(
    "gpt-4o-mini",           # 第一个备选
    "claude-3-haiku",        # 第二个备选
)
```

### 3.5 ModelFallbackMiddleware 与 ModelRetryMiddleware 的优先级

Middleware 按照**洋葱模型**包裹模型调用，列表中的顺序决定执行层次。

**推荐：Fallback 在外，Retry 在内**

```python
agent = create_agent(
    model="gpt-4o",
    middleware=[
        ModelFallbackMiddleware("gpt-4o-mini", "claude-3-haiku"),  # 外层
        ModelRetryMiddleware(max_retries=3),                       # 内层
    ],
)
```

失败时的流程：
1. 内层 Retry 先对当前模型重试 3 次（带退避）
2. 如果重试耗尽，外层 Fallback 切换到下一个备选模型
3. 切换后，内层 Retry 再对新模型重试 3 次
4. 依次类推，直到成功或所有备选都耗尽

### 3.6 SummarizationMiddleware — 对话摘要

对话过长时自动压缩摘要。

```python
from langchain.agents.middleware import SummarizationMiddleware

middleware = SummarizationMiddleware(
    model="gpt-4o-mini",                              # 用于生成摘要的模型
    trigger=[("tokens", 4000), ("messages", 50)],     # 触发条件
    keep=("messages", 20),                            # 保留最近 20 条消息
    summary_prompt="请总结以下对话的关键信息：",         # 自定义提示词
)
```

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `model` | `str \| BaseChatModel` | 必填 | 用于生成摘要的 LLM |
| `trigger` | `ContextSize \| TriggerClause \| list \| None` | `None` | 触发摘要的条件 |
| `keep` | `ContextSize` | `('messages', 20)` | 摘要后保留的历史内容 |
| `token_counter` | `TokenCounter` | `count_tokens_approximately` | 计算 token 数的函数 |
| `summary_prompt` | `str` | `DEFAULT_SUMMARY_PROMPT` | 生成摘要的提示词模板 |
| `trim_tokens_to_summarize` | `int \| None` | 默认限制 | 准备摘要时保留的最大 token 数 |

**触发条件类型：**
- `("messages", N)` — 消息数量达到 N
- `("tokens", N)` — token 数达到 N
- `("fraction", F)` — 占模型最大输入的 F 比例

**工作原理：**
1. 每次模型调用前检查是否触发（token 数或消息数达到阈值）
2. 将消息列表分为旧消息和保留消息
3. 把旧消息 + summary_prompt 发给摘要模型生成摘要
4. 用一条 **HumanMessage**（不是 SystemMessage）替换旧消息
5. 摘要消息 + 保留的最近消息 → 发给主模型

### 3.7 HumanInTheLoopMiddleware — 人工审批

暂停 Agent 执行，等待人工审批、编辑或拒绝工具调用。

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langchain.agents.middleware.human_in_the_loop import InterruptOnConfig

middleware = HumanInTheLoopMiddleware(
    interrupt_on={
        "read_file": False,        # 不需要审批
        "edit_file": True,         # 默认审批（approve/edit/reject）
        "send_email": InterruptOnConfig(
            allowed_decisions=["approve", "reject"]  # 只能批准或拒绝
        ),
    },
    description_prefix="⚠️ 此操作需要人工确认",
)
```

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `interrupt_on` | `dict[str, bool \| InterruptOnConfig]` | ✅ | 指定哪些工具需要人工审批 |
| `description_prefix` | `str` | ❌ | 审批提示的前缀文本 |

**决策类型（DecisionType）：**

```python
DecisionType = Literal["approve", "edit", "reject", "respond"]
```

| 决策 | 含义 | 典型场景 |
|---|---|---|
| `approve` | 按原样执行工具 | 审批高风险操作 |
| `edit` | 修改参数后执行 | 调整工具调用细节 |
| `reject` | 拒绝执行 | 否定副作用操作 |
| `respond` | 人类直接作为工具输出 | 回答 `ask_user` 类工具的问题 |

**`respond` 说明：** 当工具本身就是用来向用户提问的（如 `ask_user`），`respond` 让人类的回复直接成为工具的执行结果，工具本身不会真正执行。

---

## 4. 其他常用 Middleware

| Middleware | 用途 |
|---|---|
| `ModelCallLimitMiddleware` | 限制模型调用次数，防止无限循环 |
| `ToolCallLimitMiddleware` | 限制工具调用次数 |
| `PIIMiddleware` | 检测和脱敏个人敏感信息 |
| `TodoListMiddleware` | 给 Agent 添加任务规划能力（自带 `write_todos` 工具） |
| `LLMToolSelectorMiddleware` | 用 LLM 预选相关工具，减少 token 消耗 |
| `ProviderToolSearchMiddleware` | 将工具搜索延迟到 provider 服务端 |
| `ShellToolMiddleware` | 提供持久化 shell 会话执行系统命令 |
| `FilesystemMiddleware` | 给 Agent 提供文件系统 |
| `SubAgentMiddleware` | 支持生成子 Agent，隔离上下文 |
| `ContextEditingMiddleware` | 裁剪旧的工具输出来节省 token |
| `LLMToolEmulator` | 用 LLM 模拟工具执行，用于测试 |

---

## 5. 自定义 Middleware

### 5.1 函数式（装饰器风格）

```python
from langchain.agents.middleware import before_model, after_model
from langchain.messages import AIMessage

@before_model(can_jump_to=["end"])
def check_message_limit(state, runtime) -> dict | None:
    """限制消息数量"""
    if len(state["messages"]) >= 50:
        return {
            "messages": [AIMessage("消息数量已达上限")],
            "jump_to": "end"
        }
    return None

@after_model
def log_response(state, runtime):
    """记录模型响应"""
    print(f"Model returned: {state['messages'][-1].content}")
```

### 5.2 类式

```python
from langchain.agents.middleware import AgentMiddleware

class MyMiddleware(AgentMiddleware):
    def before_model(self, state, runtime):
        # 在模型调用前执行
        ...

    async def abefore_model(self, state, runtime):
        # 异步版本
        ...

    def wrap_tool_call(self, request, handler):
        try:
            return handler(request)
        except Exception as e:
            return ToolMessage(content=f"Error: {e}", tool_call_id=request.tool_call["id"])
```

### 5.3 can_jump_to 跳转控制

```python
# 提前结束
@before_model(can_jump_to=["end"])
def check_limit(state, runtime):
    if condition:
        return {"jump_to": "end"}
    return None

# 跳过模型，直接执行工具
@before_model(can_jump_to=["tools"])
def force_tool_call(state, runtime):
    if needs_forced_action(state):
        return {"jump_to": "tools"}
    return None

# 多个跳转目标
@before_model(can_jump_to=["end", "tools"])
def smart_router(state, runtime):
    if should_end(state):
        return {"jump_to": "end"}
    elif should_force_tool(state):
        return {"jump_to": "tools"}
    return None
```

### 5.4 跳过模型调用，直接返回工具输出

类似 OpenAI 的 `StopAtTools`，可以使用 `after_tool` + `jump_to="end"`：

```python
from langchain.agents.middleware import after_tool, hook_config

@after_tool
@hook_config(can_jump_to=["end"])
def return_tool_output_directly(state, runtime):
    """工具输出直接作为最终响应，不再调用模型"""
    return {"jump_to": "end"}
```

---

## 6. 便捷装饰器

### 6.1 @dynamic_prompt

用于动态生成系统提示词：

```python
from langchain.agents.middleware import dynamic_prompt

@dynamic_prompt
def personalized_prompt(state, runtime) -> str:
    user = runtime.context
    return f"""你是一个智能助手。
当前用户：{user.name}
用户角色：{user.role}
请根据用户的角色和背景，提供专业且个性化的回答。"""

agent = create_agent(
    model="gpt-4o",
    tools=[...],
    middleware=[personalized_prompt],
)
```

### 6.2 @hook_config

用于类式 middleware 中声明 hook 的控制流权限：

```python
class MyMiddleware(AgentMiddleware):
    @before_model
    @hook_config(can_jump_to=["end"])  # 声明允许跳转到 end
    def check_something(self, state, runtime):
        if condition:
            return {"jump_to": "end"}
        return None
```

**为什么需要 `@hook_config`？**
- 函数式 middleware：直接用 `@before_model(can_jump_to=["end"])`
- 类式 middleware：hook 是类方法，不能直接传参，需要 `@hook_config` 装饰器

---

## 7. 类型注解说明

### `tuple[type[Exception], ...]`

表示包含任意数量异常类型的元组：

```python
# 合法的例子
errors: tuple[type[Exception], ...] = ()                    # 空元组
errors = (ValueError,)                                      # 一个异常类型
errors = (ValueError, TypeError, ConnectionError)           # 多个异常类型
```

---

## 8. 参考链接

- [Middleware Overview](https://docs.langchain.com/oss/python/langchain/middleware/overview)
- [Prebuilt Middleware](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [Custom Middleware](https://docs.langchain.com/oss/python/langchain/middleware/custom)
- [AgentMiddleware Reference](https://reference.langchain.com/python/langchain/agents/middleware/types/AgentMiddleware)
- [before_model Reference](https://reference.langchain.com/python/langchain/agents/middleware/types/before_model)
- [hook_config Reference](https://reference.langchain.com/python/langchain/agents/middleware/types/hook_config)
- [Human-in-the-loop Docs](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)
