# Guardrails 与人工审批（Guardrails and Human Review）

## 四种控制手段

| 控制手段 | 检查什么 | 谁做决定 | 触发时机 |
|---------|---------|---------|---------|
| **Input guardrails** | 用户输入 | 程序自动 | 主模型运行**之前** |
| **Output guardrails** | Agent 输出 | 程序自动 | 返回用户**之前** |
| **Tool guardrails** | 工具参数/结果 | 程序自动 | 工具调用**前后** |
| **Human-in-the-loop** | 工具执行前暂停 | **人来决定** | 工具执行**之前** |

## 选择合适的控制手段

| 使用场景 | 选择 |
|---------|------|
| 在主模型运行前阻止不合规的用户请求 | Input guardrails |
| 在最终输出返回前验证或脱敏 | Output guardrails |
| 检查函数工具的参数或结果 | Tool guardrails |
| 在取消订单、编辑数据、执行 shell 命令等副作用操作前暂停 | Human-in-the-loop approvals |

---

## Input Guardrails（输入护栏）

在主 Agent 处理用户输入**之前**，先让一个守卫 Agent 检查输入是否合规。

```python
from pydantic import BaseModel
from agents import (
    Agent, GuardrailFunctionOutput, InputGuardrailTripwireTriggered,
    RunContextWrapper, Runner, TResponseInputItem, input_guardrail,
)

class MathHomeworkOutput(BaseModel):
    is_math_homework: bool
    reasoning: str

guardrail_agent = Agent(
    name="Homework check",
    instructions="Detect whether the user is asking for math homework help.",
    output_type=MathHomeworkOutput,
)

@input_guardrail
async def math_guardrail(
    ctx: RunContextWrapper[None],
    agent: Agent,
    input: str | list[TResponseInputItem],
) -> GuardrailFunctionOutput:
    result = await Runner.run(guardrail_agent, input, context=ctx.context)
    return GuardrailFunctionOutput(
        output_info=result.final_output,
        tripwire_triggered=result.final_output.is_math_homework,
    )

agent = Agent(
    name="Customer support",
    instructions="Help customers with support questions.",
    input_guardrails=[math_guardrail],
)

try:
    await Runner.run(agent, "Can you solve 2x + 3 = 11 for me?")
except InputGuardrailTripwireTriggered:
    print("Guardrail blocked the request.")
```

---

## Output Guardrails（输出护栏）

在主 Agent 生成回复后、返回给用户**之前**进行检查。机制与 Input Guardrails 相同。

```python
from agents import output_guardrail, OutputGuardrailTripwireTriggered

@output_guardrail
async def sensitive_filter(ctx, agent, output) -> GuardrailFunctionOutput:
    contains_sensitive = "信用卡号" in output
    return GuardrailFunctionOutput(
        output_info={"contains_sensitive": contains_sensitive},
        tripwire_triggered=contains_sensitive,
    )

agent = Agent(
    name="Support",
    instructions="...",
    output_guardrails=[sensitive_filter],
)

try:
    result = await Runner.run(agent, "用户的请求")
except OutputGuardrailTripwireTriggered:
    print("输出被拦截")
```

---

## Tool Guardrails（工具护栏）

检查工具的**参数**（执行前）和**结果**（执行后），挂在工具定义上，不是 Agent 上。

```python
from agents import (
    function_tool, tool_input_guardrail, tool_output_guardrail,
    ToolGuardrailFunctionOutput,
)

@tool_input_guardrail
async def check_amount(ctx, tool_args) -> ToolGuardrailFunctionOutput:
    if tool_args.get("amount", 0) > 10000:
        return ToolGuardrailFunctionOutput.reject_content("金额超过上限")
    return ToolGuardrailFunctionOutput.allow()

@tool_output_guardrail
async def check_result(ctx, tool_output) -> ToolGuardrailFunctionOutput:
    if "error" in str(tool_output).lower():
        return ToolGuardrailFunctionOutput.reject_content("工具返回了错误")
    return ToolGuardrailFunctionOutput.allow()

@function_tool(
    tool_input_guardrails=[check_amount],
    tool_output_guardrails=[check_result],
)
async def transfer_money(amount: float, to_account: str) -> str:
    return f"Transferred {amount} to {to_account}"
```

**便捷方法：**
- `ToolGuardrailFunctionOutput.allow()` — 放行
- `ToolGuardrailFunctionOutput.reject_content("原因")` — 拒绝并反馈原因给模型
- `ToolGuardrailFunctionOutput.raise_exception(...)` — 硬失败

---

## Human-in-the-loop（人工审批）

### 核心机制

使用 `needs_approval=True` 标记工具，SDK 会在工具执行前产生 **interruption（中断）**。

```python
from agents import function_tool

@function_tool(needs_approval=True)
async def cancel_order(order_id: int) -> str:
    return f"Canceled {order_id}"
```

### 审批生命周期

```
1. Run 记录一个审批中断，而不是执行工具
2. Result 返回 interruptions + 可恢复的 state
3. 你的应用批准或拒绝待处理的审批项
4. 从 state 恢复同一个 run（不是新的用户对话）
```

### 代码实现

```python
result = await Runner.run(agent, input="Cancel order 123")

if result.interruptions:
    state = result.to_state()
    for interruption in result.interruptions:
        print(f"工具调用待审批: {interruption.tool_name}")
        user_input = input("批准？(y/n): ")
        if user_input.lower() == "y":
            state.approve(interruption)
        else:
            print("已拒绝")
    result = await Runner.run(agent, state)

print(result.final_output)
```

### 关键区别

| 模式 | 谁负责审批 | 效果 |
|------|-----------|------|
| Debug 模式 | Trace Viewer UI（在 Runner 内部拦截） | 自动弹出审批界面 |
| Normal run + `input()` | 你的代码里手动等待 | 暂停等用户输入 |
| Normal run + 自动 `approve()` | `state.approve()` 直接调用 | 自动批准，不停顿 |

### 延迟审批

如果审批不能立刻做（如需要邮件审批、走工单），可以把 state 序列化存储，之后再恢复：

```python
# 存起来
import json
state_data = state.to_json()
# 保存到数据库...

# 以后恢复
state = RunState.from_json(state_data)
state.approve(interruption)
result = await Runner.run(agent, state)
```

流式（streaming）场景也是同一套 state 模型，没有额外的审批系统。

---

## Shell 工具的审批

### 本地 Shell（支持审批）

```python
from agents import ShellTool

shell_tool = ShellTool(needs_approval=True)
```

也支持 `on_approval` 回调做程序化审批：

```python
shell_tool = ShellTool(on_approval=my_approval_callback)
```

### 托管沙箱（不支持审批）

> **"Hosted shell environments do not support needs_approval or on_approval"**

托管沙箱环境目前不支持审批机制，这是环境限制。

---

## 工作流边界（重要）

Agent 级别的 guardrails **不是每个 Agent 都会执行**：

| Guardrail 类型 | 执行范围 |
|---|---|
| **Input guardrails** | 只在**链中第一个 Agent** 上执行 |
| **Output guardrails** | 只在**产出最终输出的 Agent** 上执行 |
| **Tool guardrails** | 在它们**所挂载的工具**被调用时执行 |

**建议：** 如果需要在每个工具调用时都做检查（如 manager 模式下多个 Agent 各自调工具），不要只依赖 Agent 级别的 input/output guardrails，应该用 **tool guardrails** 挂在工具上——不管哪个 Agent 调用，检查都会执行。

---

## 网络安全场景

对于授权的安全测试工作流，用 tool guardrails + approval interruptions 确保操作在授权范围内：

- 检查目标、操作类型、工具参数、调用者身份、授权时间窗口
- 拒绝：范围外主机、凭据窃取、持久化植入、数据外泄、破坏性修改、生产环境访问、绕过策略
- 模糊/高风险操作暂停等人工批准
- 审批超时或不可用时**默认拒绝**（fail closed）
- Responses API 和 Agents SDK **不会自动继承 Codex Auto-review**，需要自己实现审批逻辑
