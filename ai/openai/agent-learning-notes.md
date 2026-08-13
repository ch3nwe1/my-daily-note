# AI Agent 学习笔记

> 日期：2026-08-13 ~ 2026-08-14

---

## 一、AI Agent 基础概念

### 1.1 什么是 AI Agent

AI Agent（智能体）是一种能够**自主感知环境、做出决策并执行行动**来实现目标的软件系统。

**核心组成**：

| 组件 | 说明 |
|------|------|
| 感知（Perception） | 接收外部输入，如用户指令、API 返回、文件变化等 |
| 推理/规划（Reasoning/Planning） | 基于大模型理解意图、分解任务、制定步骤 |
| 记忆（Memory） | 短期记忆（上下文窗口）+ 长期记忆（向量数据库、文件等） |
| 工具（Tools） | 可调用的外部能力：搜索、代码执行、API 调用等 |
| 行动（Action） | 将决策转化为具体操作，输出结果 |

### 1.2 Agent 与普通 ChatBot 的区别

```
普通 ChatBot：
用户提问 → 模型回答（单轮，无工具，无状态管理）

AI Agent：
用户目标 → Agent 自主拆解任务
         → 调用工具执行
         → 观察结果
         → 决定下一步
         → 循环直到完成目标
```

### 1.3 关键特征

- **自主性（Autonomy）**：给定目标后，能自己决定该做什么
- **迭代性（Iterative）**：通过"思考→行动→观察"的循环持续推进
- **工具使用（Tool Use）**：能调用外部系统扩展能力边界
- **目标驱动（Goal-Oriented）**：围绕一个目标工作

---

## 二、主流 Agent 框架对比

### 2.1 AutoGPT vs CrewAI

| 维度 | AutoGPT | CrewAI |
|------|---------|--------|
| 核心理念 | 单个超级 Agent，自主完成一切 | 多个角色 Agent 组队协作 |
| 类比 | 一个全能员工 | 一个团队（产品经理+开发+测试） |
| 控制方式 | Agent 自己决定下一步 | 预定义流程和角色分工 |
| 适用场景 | 个人实验、探索性任务 | 企业级工作流、需要稳定输出 |

**一句话总结**：AutoGPT 是"一个聪明的自由职业者"，CrewAI 是"一支管理有序的项目团队"。

### 2.2 LangChain 构建 Agent

LangChain 可以构建 Agent，有三条路径：

| 路径 | 状态 | 适用场景 |
|------|------|----------|
| AgentExecutor（langchain.agents） | ⚠️ Legacy | 老项目维护 |
| LangGraph | ✅ 官方推荐 | 需要精细控制流程的 Agent |
| LangChain + LCEL | ✅ 可用 | 简单的链式调用 |

### 2.3 框架选型建议

| 需求 | 推荐框架 |
|------|----------|
| 快速原型、不想写代码 | AutoGPT / Dify / Coze |
| 需要多步循环 + 精细控制 | LangGraph |
| 多角色团队协作 | CrewAI |
| 只用 OpenAI 模型 | OpenAI Agents SDK |
| 用 Claude 模型 | Claude Agent SDK |

---

## 三、OpenAI Agents SDK 实战

### 3.1 基本用法

```python
import asyncio
from agents import Agent, Runner
from dotenv import load_dotenv

load_dotenv(verbose=True, dotenv_path='../.env')

agent = Agent(
    name="History tutor",
    instructions="You answer history questions clearly and concisely.",
    model="deepseek-v4-pro"
)

async def main() -> None:
    result = await Runner.run(agent, "When did the Roman Empire fall?")
    print(result.final_output)

if __name__ == '__main__':
    asyncio.run(main())
```

### 3.2 Tracing 报错处理

运行后出现 `[non-fatal] Tracing client error 401`：

- **原因**：SDK 默认将追踪数据发送到 OpenAI 服务器，但使用的是 DeepSeek API Key，认证失败
- **影响**：不影响主流程，是 non-fatal 错误
- **解决方案**：
  ```python
  from agents.tracing import set_trace_processors
  set_trace_processors([])  # 关闭追踪
  ```
  或在 `.env` 中添加：
  ```bash
  AGENTS_DISABLE_TRACING=true
  ```

### 3.3 LangSmith 追踪替代方案

| 工具 | 免费额度 | 特点 |
|------|----------|------|
| LangSmith | 5,000 traces/月 | 和 LangChain 生态深度集成 |
| Arize Phoenix | 开源自部署 | 完全免费，无追踪量限制 |
| Langfuse | 开源自部署 | 社区活跃，支持多框架 |

---

## 四、为 Agent 添加工具

### 4.1 基本结构

```python
from agents import Agent, Runner, function_tool

@function_tool
def get_weather(city: str) -> str:
    """获取指定城市的天气信息"""
    return f"{city}今天晴，25°C"

agent = Agent(
    name="Weather Assistant",
    instructions="你是一个天气助手。",
    tools=[get_weather],
)
```

### 4.2 工具定义方式

**方式一：`@function_tool` 装饰器（最常用）**

```python
@function_tool
def calculate(expression: str) -> str:
    """计算数学表达式"""
    return str(eval(expression))
```

**方式二：异步工具**

```python
@function_tool
async def fetch_url(url: str) -> str:
    """抓取网页内容"""
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
        return resp.text[:1000]
```

### 4.3 关键注意事项

| 要点 | 说明 |
|------|------|
| docstring 必须写 | Agent 靠 docstring 理解工具功能 |
| 参数类型要明确 | 用 Python 类型标注，自动转 JSON Schema |
| 返回值是字符串 | 复杂数据用 `json.dumps()` 转换 |
| 工具名和描述用英文 | 传给模型效果更好 |

---

## 五、Agent 配置

### 5.1 设置温度

```python
from agents import Agent, ModelSettings

agent = Agent(
    name="Creative Writer",
    instructions="你是一个创意写作助手。",
    model="deepseek-v4-pro",
    model_settings=ModelSettings(
        temperature=0.7,
        top_p=0.9,
        max_tokens=2048,
    ),
)
```

**温度推荐**：

| 场景 | 温度值 |
|------|--------|
| 精确问答 / 代码生成 | 0.0 ~ 0.2 |
| 日常对话 / 翻译 | 0.3 ~ 0.5 |
| 创意写作 / 头脑风暴 | 0.7 ~ 1.0 |

### 5.2 统一设置模型

**方法一：环境变量（推荐）**

```bash
# .env 文件
OPENAI_DEFAULT_MODEL=deepseek-v4-pro
```

代码中不需要写 `model` 参数，所有 Agent 自动使用。

**优先级（从高到低）**：

```
Agent 指定的 model         ← 最高
RunConfig 指定的 model
OPENAI_DEFAULT_MODEL 环境变量
gpt-4o（SDK 内置默认值）    ← 最低
```

---

## 六、多 Agent 协作：专家分工模式

### 6.1 概念

```
用户问题
   ↓
┌──────────────┐
│   路由 Agent   │  ← 判断问题类型，分发给专家
│  （Router）    │
└──────┬───────┘
       │
   ┌───┼───┬──────────┐
   ↓   ↓   ↓          ↓
专家A 专家B 专家C     专家D
```

- **Specialist Agents（专家 Agent）**：各自只负责某个领域
- **Router（路由 Agent）**：接收输入，判断该交给哪个专家
- **Handoff（交接）**：路由 Agent 把任务"移交"给专家的过程

### 6.2 代码示例

```python
from agents import Agent, Runner

math_agent = Agent(
    name="Math Specialist",
    instructions="你只回答数学问题。",
    handoff_description="处理数学问题，如代数、几何、微积分",
)

history_agent = Agent(
    name="History Specialist",
    instructions="你只回答历史问题。",
    handoff_description="处理历史问题，如朝代、战争、人物",
)

router = Agent(
    name="Router",
    instructions="根据问题类型移交给合适的专家。",
    handoffs=[math_agent, history_agent],
)
```

### 6.3 移交规则定义方式

| 方式 | 位置 | 适合场景 |
|------|------|----------|
| `instructions` 里写路由规则 | Router Agent | 简单项目 |
| `handoff_description` | 每个专家 Agent | 多专家、需可维护性 |
| `handoff()` 函数 | 单独定义 | 需自定义移交行为、回调 |

**推荐用 `handoff_description`**，符合"谁做事谁描述"的原则。

---

## 七、其他注意事项

### 7.1 LaTeX 公式显示问题

Agent 输出含 LaTeX 公式时，终端会显示原始源码（带反斜杠 `\`），因为终端不支持渲染。

**解决方案**：

- 让 Agent 用纯文本格式回答：
  ```python
  instructions="回答时使用纯文本，不要用 LaTeX 数学公式。"
  ```
- 输出到支持 LaTeX 的环境（Jupyter、Obsidian）
- 保存为 Markdown 文件查看

---

## 参考链接

- [OpenAI Agents SDK 官方文档](https://openai.github.io/openai-agents-python/)
- [Handoffs 文档](https://openai.github.io/openai-agents-python/handoffs/)
- [Models 文档](https://openai.github.io/openai-agents-python/models/)
- [LangSmith 定价](https://www.langchain.com/pricing)
