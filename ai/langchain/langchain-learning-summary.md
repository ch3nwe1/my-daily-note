# LangChain 学习笔记总结

**日期**: 2026-08-18  
**学习主题**: LangChain 框架核心概念和 API

---

## 目录

1. [模型使用方式](#1-模型使用方式)
2. [模型参数配置](#2-模型参数配置)
3. [多轮对话与状态管理](#3-多轮对话与状态管理)
4. [工具调用机制](#4-工具调用机制)
5. [结构化输出](#5-结构化输出)
6. [Model Profile 系统](#6-model-profile-系统)
7. [性能优化](#7-性能优化)
8. [运行时配置](#8-运行时配置)
9. [动态模型选择](#9-动态模型选择)

---

## 1. 模型使用方式

### 两种使用模式

```python
# 方式 1：与 Agent 一起使用
agent = create_agent(model=model, tools=[...])

# 方式 2：独立使用
response = model.invoke("你好")
```

**关键点**: 同一个模型接口在两种场景下都适用，提供灵活性。

---

## 2. 模型参数配置

### 核心参数

| 参数 | 类型 | 默认值 | 作用 |
|------|------|--------|------|
| `max_retries` | int | 6 | 最大重试次数 |
| `timeout` | float | - | 请求超时时间（秒） |

### 重试机制

```python
model = init_chat_model(
    "openai:gpt-4",
    max_retries=10,  # 重试 10 次
    timeout=30       # 超时 30 秒
)
```

**自动重试的错误**:
- 网络错误
- 429（速率限制）
- 5xx（服务器错误）

**不重试的错误**:
- 401（未授权）
- 404（资源不存在）
- 其他客户端错误

**指数退避**: 1s → 2s → 4s → 8s → ...

---

## 3. 多轮对话与状态管理

### Checkpointer 机制

```python
from langgraph.checkpoint.memory import InMemorySaver

memory = InMemorySaver()

agent = create_agent(
    model=model,
    tools=[],
    checkpointer=memory  # 启用状态保存
)

# 多轮对话（使用 thread_id 标识会话）
config = {"configurable": {"thread_id": "user-123"}}

response1 = agent.invoke({"messages": "我叫张三"}, config=config)
response2 = agent.invoke({"messages": "我叫什么？"}, config=config)
# Agent 记得你叫张三
```

### Checkpointer 工作原理

```
1. 加载状态: get(thread_id) → 之前的状态
2. 执行 Agent → 产生新状态
3. 保存状态: save(thread_id, 新状态)
```

**支持的存储后端**:
- `InMemorySaver`: 内存（开发/测试）
- `PostgresSaver`: PostgreSQL（生产）
- `SqliteSaver`: SQLite
- 自定义实现

---

## 4. 工具调用机制

### bind_tools vs tools 参数

```python
# bind_tools: 模型层级，告诉 LLM 有工具
model_with_tools = model.bind_tools([search_tool])

# create_agent(tools=): Agent 层级，自动执行工具
agent = create_agent(model=model, tools=[search_tool])
```

**核心区别**:
- `bind_tools`: 只告诉模型有工具，不会自动执行
- `create_agent(tools=)`: 自动执行工具并处理结果

### 为什么 bind_tools 返回新对象？

遵循**不可变性原则**:
- 原模型不被修改
- 可以创建多个变体
- 线程安全
- 支持链式调用

```python
model_with_search = model.bind_tools([search_tool])
model_with_calculator = model.bind_tools([calculator_tool])
# 原 model 不变
```

---

## 5. 结构化输出

### with_structured_output 方法

```python
from pydantic import BaseModel

class Movie(BaseModel):
    title: str
    year: int
    director: str
    rating: float

model_with_structure = model.with_structured_output(Movie)
response = model_with_structure.invoke("介绍《盗梦空间》")
# 返回: Movie(title='盗梦空间', year=2010, ...)
```

### 实际工作流程

```
1. LangChain 提取 Movie 的 JSON Schema
2. 作为 tools 参数发送给 API（不是提示词！）
3. 模型返回 tool_calls（结构化数据）
4. LangChain 解析并转换成 Movie 对象
```

**关键点**: JSON Schema 是作为 `tools` 参数发送的，不是提示词，所以在 LangSmith 中默认看不到。

### method 参数

| method | 原理 | 可靠性 | 兼容性 |
|--------|------|--------|--------|
| `function_calling` | 工具调用 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| `json_mode` | JSON 模式 | ⭐⭐ | ⭐⭐⭐ |
| `json_schema` | 严格 Schema | ⭐⭐⭐⭐⭐ | ⭐⭐ |

**默认值**: `json_schema`（最严格，但只有新模型支持）

### include_raw 参数

```python
# 默认：只返回解析后的对象
response = model.with_structured_output(Movie).invoke("...")
# 返回: Movie(...)

# include_raw=True: 返回完整信息
response = model.with_structured_output(Movie, include_raw=True).invoke("...")
# 返回: {
#     'raw': AIMessage(...),      # 原始消息
#     'parsed': Movie(...),       # 解析后的对象
#     'parsing_error': None       # 解析错误
# }
```

### TypedDict vs Pydantic

```python
# Pydantic（推荐，有验证）
from pydantic import BaseModel

class Movie(BaseModel):
    title: str
    year: int

# TypedDict（轻量，无验证）
from typing_extensions import TypedDict, Annotated

class MovieDict(TypedDict):
    title: Annotated[str, ..., "电影标题"]
    year: Annotated[int, ..., "上映年份"]
```

**选择建议**:
- 需要验证、序列化 → Pydantic
- 简单场景、追求性能 → TypedDict
- Agent 状态 → TypedDict
- 输入/输出 → Pydantic

### Annotated 语法

```python
from typing_extensions import Annotated

# 可以放任何东西
title: Annotated[str, "描述"]
title: Annotated[str, ..., "描述"]  # ... 是占位符
title: Annotated[str, Field(description="描述")]
title: Annotated[str, {"key": "value"}, "描述"]
```

**关键点**: Annotated 只是收集元数据，具体含义由消费方（LangChain、Pydantic）决定。

---

## 6. Model Profile 系统

### 数据来源

```
models.dev（开源项目）
   ↓
LangChain 增强（profile_augmentations.toml）
   ↓
合并后的 profiles.json
```

### Profile 内容

```python
{
    "name": "gpt-4",
    "max_input_tokens": 8192,
    "tool_calling": True,
    "structured_output": True,
    "image_inputs": True,
    # ... 更多能力描述
}
```

### 实际用途

1. **自动选择方法**: 根据模型能力选择 json_schema 或 function_calling
2. **验证输入**: 检查是否支持图像、PDF 等
3. **摘要中间件**: 根据上下文窗口触发
4. **模型选择器**: 只显示兼容的模型

### 更新流程

```bash
# 开发者
langchain-model-profiles update
# 自动下载 models.dev 数据并合并

# 用户
pip install --upgrade langchain-openai
# 使用打包在包里的 profiles.json（不联网）
```

### 字典合并语法

```python
new_profile = model.profile | {"key": "value"}
# Python 3.9+ 的字典合并操作符
# 右边覆盖左边的同名键
```

---

## 7. 性能优化

### Prompt Caching（提示缓存）

**三个级别**:

1. **隐式缓存**（自动）
   - 提供商自动处理
   - 无需配置
   - OpenAI、Gemini 支持

2. **显式控制**（手动）
   - OpenAI: `prompt_cache_key`
   - Anthropic: `cache_control`
   - AWS Bedrock: `cachePoint`

3. **中间件**（Agent 最佳）
   ```python
   from langchain_anthropic.middleware import AnthropicPromptCachingMiddleware
   
   agent = create_agent(
       model="anthropic:claude-3-opus",
       middleware=[AnthropicPromptCachingMiddleware()]
   )
   ```

**最小阈值**: 通常 1024 tokens，低于此数不缓存

### Rate Limiting（速率限制）

```python
from langchain_core.rate_limiters import InMemoryRateLimiter

rate_limiter = InMemoryRateLimiter(
    requests_per_second=1.0,
    requests_per_minute=60
)

model = init_chat_model(
    "openai:gpt-4",
    rate_limiter=rate_limiter
)
```

**作用**: 自动控制请求速率，避免触发 429 错误

### Log Probabilities（对数概率）

```python
model = init_chat_model(
    "openai:gpt-4",
    logprobs=True,
    top_logprobs=5
)

response = model.invoke("法国的首都是")
# response.response_metadata["logprobs"] 包含每个 token 的概率
```

**用途**:
- 检测不确定性
- 发现幻觉
- 选择最佳完成
- 分类任务

---

## 8. 运行时配置

### RunnableConfig

```python
config = {
    "run_id": str(uuid.uuid4()),      # 唯一标识
    "run_name": "chat_user_123",      # 可读名称
    "tags": ["production", "chat"],   # 标签分类
    "metadata": {"user_id": "123"},   # 元数据
    "callbacks": [MyCallback()],      # 回调函数
    "max_concurrency": 5,             # 最大并发
    "recursion_limit": 10             # 递归限制
}

response = model.invoke("你好", config=config)
```

### 各字段作用

| 字段 | 作用 | 示例 |
|------|------|------|
| `run_id` | 唯一标识调用 | UUID |
| `run_name` | 可读名称 | `chat_user_123` |
| `tags` | 分类标签 | `["production", "premium"]` |
| `metadata` | 追踪元数据 | `{"user_id": "123"}` |
| `callbacks` | 回调函数 | 日志、监控 |

### configurable_fields（运行时可配置）

```python
# 创建可配置模型（占位符）
configurable_model = init_chat_model().configurable_fields(
    model="model_name",
    model_provider="provider"
)

# 调用时动态指定
response = configurable_model.invoke(
    "你好",
    config={
        "configurable": {
            "model": "gpt-4",
            "model_provider": "openai"
        }
    }
)
```

**与固定模型的区别**:
- 固定模型: `init_chat_model("openai:gpt-4")` → 不能改
- 可配置模型: 运行时可以切换不同模型

---

## 9. 动态模型选择

### @wrap_model_call 装饰器

```python
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse

@wrap_model_call
def dynamic_model_selection(request: ModelRequest, handler) -> ModelResponse:
    """根据对话复杂度选择模型"""
    
    # 1. 分析请求
    message_count = len(request.state["messages"])
    
    # 2. 选择模型
    if message_count > 10:
        model = advanced_model
    else:
        model = basic_model
    
    # 3. 修改请求并传递给 handler
    return handler(request.override(model=model))

# 应用到 Agent
agent = create_agent(
    model=basic_model,
    tools=tools,
    middleware=[dynamic_model_selection]
)
```

### 中间件模式

```
请求流程:
用户调用 → 中间件 1 → 中间件 2 → ... → 实际模型调用
            ↓           ↓                      ↓
         handler      handler              返回结果
```

**handler 是什么**:
- 中间件链中的下一个处理器
- 可以是另一个中间件，也可以是实际的模型调用
- 接收修改后的 request，返回 response

### 实际应用场景

1. **成本优化**: 简单任务用便宜模型
2. **任务路由**: 代码任务用 Claude，推理任务用 GPT-4
3. **用户偏好**: 根据用户选择模型
4. **负载均衡**: 根据系统负载选择

### 成本优化效果

```
没有动态选择（都用 gpt-4）: $300/天
有动态选择: $152/天
节省: 49%！
```

---

## 关键概念总结

### API 调用 vs 普通聊天

| 功能 | Web 聊天界面 | API 调用 |
|------|------------|---------|
| 发送消息 | ✅ | ✅ |
| 定义工具 | ❌ | ✅ |
| 结构化输出 | ❌ | ✅ |
| 控制温度 | ❌ | ✅ |
| 查看 token | ❌ | ✅ |

### 三层架构

```
┌─────────────────────────────────────┐
│  Agent 框架（LangChain）            │ ← 实现智能体逻辑
│  - 工具执行、状态管理、循环          │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  大模型 API                         │ ← 提供结构化输出能力
│  - 生成文本、tool_calls、JSON       │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│  模型推理引擎                       │ ← 纯粹的文本生成
│  - 预测下一个 token                 │
└─────────────────────────────────────┘
```

### 重要原则

1. **不可变性**: bind_tools 返回新对象，不修改原对象
2. **中间件模式**: 通过 handler 链接多个处理器
3. **配置分离**: 初始化参数（模型行为）vs RunnableConfig（运行时行为）
4. **自动适配**: Model Profile 让应用自动适配不同模型的能力

---

## 常见误区

### ❌ 误区 1: 大模型内置了智能体
**✅ 正确理解**: Agent 是框架层的概念，模型只提供生成能力

### ❌ 误区 2: JSON Schema 作为提示词发送
**✅ 正确理解**: 作为 `tools` 参数发送，所以日志中看不到

### ❌ 误区 3: LangChain 运行时读取 models.dev
**✅ 正确理解**: 构建时下载并打包，运行时读取本地文件

### ❌ 误区 4: 直接修改安装包的 profiles.json
**✅ 正确理解**: 运行时动态添加或提交 PR 到官方

---

## 实用代码片段

### 完整的多轮对话 Agent

```python
from langchain.chat_models import init_chat_model
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver
from langchain.tools import tool

@tool
def search(query: str):
    """搜索工具"""
    return f"搜索结果: {query}"

# 创建模型
model = init_chat_model(
    "openai:gpt-4",
    max_retries=10,
    timeout=60
)

# 创建记忆
memory = InMemorySaver()

# 创建 Agent
agent = create_agent(
    model=model,
    tools=[search],
    checkpointer=memory
)

# 多轮对话
config = {"configurable": {"thread_id": "user-123"}}

response1 = agent.invoke(
    {"messages": "我叫张三"},
    config=config
)

response2 = agent.invoke(
    {"messages": "我叫什么？"},
    config=config
)
# Agent 记得你叫张三
```

### 带追踪的生产环境调用

```python
import uuid
from datetime import datetime

config = {
    "run_id": str(uuid.uuid4()),
    "run_name": f"prod_chat_{datetime.now().strftime('%Y%m%d_%H%M%S')}",
    "tags": ["production", "user-facing"],
    "metadata": {
        "user_id": "user_123",
        "session_id": "session_abc"
    },
    "callbacks": [MyCallbackHandler()]
}

response = model.invoke("问题", config=config)
```

### 动态模型选择

```python
@wrap_model_call
def smart_model_router(request: ModelRequest, handler) -> ModelResponse:
    """智能模型路由"""
    
    prompt = request.state["messages"][-1].content
    
    # 根据内容选择模型
    if "代码" in prompt or "编程" in prompt:
        model = init_chat_model("anthropic:claude-3-opus")
    elif len(prompt) < 100:
        model = init_chat_model("openai:gpt-3.5-turbo")
    else:
        model = init_chat_model("openai:gpt-4")
    
    return handler(request.override(model=model))

agent = create_agent(
    model=init_chat_model("openai:gpt-4"),
    middleware=[smart_model_router]
)
```

---

## 学习资源

- **LangChain 官方文档**: https://python.langchain.com/
- **LangSmith**: https://smith.langchain.com/
- **models.dev**: https://models.dev/
- **LangGraph**: https://github.com/langchain-ai/langgraph

---

## 下一步学习方向

1. **深入 LangGraph**: 学习状态图、节点、边的概念
2. **高级中间件**: 实现自定义的中间件逻辑
3. **多 Agent 协作**: 学习如何协调多个 Agent
4. **性能优化**: 深入研究缓存、批处理、并发控制
5. **生产部署**: 学习如何部署和监控 LangChain 应用

---

**笔记整理**: 陈伟  
**最后更新**: 2026-08-18
