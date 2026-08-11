# Responses API 与 Chat Completions API 对比

## 概述

OpenAI 推出了 **Responses API** 作为新的主要接口，统一了 **Chat Completions API** 和 **Assistants API** 的能力。这是 OpenAI API 的未来方向。

---

## 一、多轮对话管理

### Chat Completions — 手动管理历史

```python
messages = [
    {"role": "system", "content": "你是一个助手"},
    {"role": "user", "content": "我叫张三"}
]

response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages
)

# 必须手动拼接历史
messages.append(response.choices[0].message)
messages.append({"role": "user", "content": "我叫什么？"})

response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages   # 每次都传完整历史
)
```

### Responses API — 三种方式管理上下文

**方式1：`previous_response_id`（让 OpenAI 管理）**

```python
res1 = client.responses.create(
    model="gpt-4o",
    input="我叫张三",
    instructions="你是一个助手"  # 每次要重新传
)

res2 = client.responses.create(
    model="gpt-4o",
    input="我叫什么？",
    previous_response_id=res1.id,  # 自动带上历史
    instructions="你是一个助手"     # 不会自动带，要重新传
)
```

**方式2：手动传 Items（自己控制上下文）**

```python
input_items = [
    {"role": "user", "content": "我叫张三"},
    {"role": "assistant", "content": "你好张三"},
    {"role": "user", "content": "我叫什么？"}
]

response = client.responses.create(
    model="gpt-4o",
    input=input_items   # 自己决定传哪些历史
)
```

**方式3：Conversations API（服务端持久化）**

```python
# 创建持久化对话对象
conversation = client.conversations.create()

# 每次传同一个 conversation id，历史自动积累
res1 = client.responses.create(
    model="gpt-4o",
    conversation=conversation.id,
    input="我叫张三"
)

res2 = client.responses.create(
    model="gpt-4o",
    conversation=conversation.id,  # 同一个 id
    input="我叫什么？"
)
```

> **注意**：Conversations API 只有 OpenAI 官方支持，DeepSeek、OpenRouter 等第三方不支持。

---

## 二、输出读取方式

| | Chat Completions | Responses API |
|---|---|---|
| 读取文本 | `response.choices[0].message.content` | `response.output_text` |
| 完整输出 | `response.choices` | `response.output`（包含多种 Item 类型） |

---

## 三、函数/工具定义

### Chat Completions — 外部标签

```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "parameters": {
      "city": {"type": "string"}
    }
  }
}
```

- `type` 和 `function` 分开，函数细节嵌套在 `function` 字段里
- 默认非严格模式（non-strict）

### Responses API — 内部标签

```json
{
  "type": "function",
  "name": "get_weather",
  "parameters": {
    "city": {"type": "string"}
  }
}
```

- 字段平铺，没有 `function` 包裹层
- 默认尝试严格模式（strict），失败则自动降级为非严格
- 想保持非严格需显式设置 `"strict": false`

---

## 四、结构化输出配置

### Chat Completions

```json
{
  "model": "gpt-4o",
  "messages": [...],
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "user_info",
      "schema": {...}
    }
  }
}
```

### Responses API

```json
{
  "model": "gpt-4o",
  "input": "...",
  "text": {
    "format": {
      "type": "json_schema",
      "name": "user_info",
      "schema": {...}
    }
  }
}
```

配置从顶层的 `response_format` 移到了 `text.format`。

---

## 五、流式输出（Streaming）

### Chat Completions — 统一结构的增量块

```json
{"choices": [{"delta": {"content": "你"}}]}
{"choices": [{"delta": {"content": "好"}}]}
```

```python
for chunk in stream:
    content = chunk.choices[0].delta.content
    if content:
        print(content, end="")
```

### Responses API — 带类型的 SSE 事件

```
event: response.created
data: {"type": "response.created", "response": {...}}

event: response.output_text.delta
data: {"type": "response.output_text.delta", "delta": "你"}

event: response.output_text.delta
data: {"type": "response.output_text.delta", "delta": "好"}

event: response.function_call_arguments.done
data: {"type": "response.function_call_arguments.done", ...}

event: response.completed
data: {"type": "response.completed", "response": {...}}
```

```python
for event in stream:
    if event.type == "response.output_text.delta":
        print(event.delta, end="")
    elif event.type == "response.function_call_arguments.done":
        execute_function(event.arguments)
    elif event.type == "response.completed":
        handle_complete(event.response)
```

---

## 六、原生工具（Native Tools）

Chat Completions 时代，所有工具都需要自己定义函数、自己实现逻辑。

Responses API 提供 OpenAI **内置工具**，开箱即用：

| 原生工具 | 作用 |
|---------|------|
| `web_search` | 联网搜索 |
| `file_search` | 文件检索 |
| `computer_use` | 操控电脑（截屏、点击、输入） |

---

## 七、函数调用中 call_id 的要求

模型调用函数时会返回 `call_id`，提交结果时**必须带上对应的 `call_id`**：

```json
// 模型返回的函数调用
{
  "type": "function_call",
  "call_id": "call_abc123",
  "name": "get_weather",
  "arguments": "{\"city\": \"北京\"}"
}

// 提交结果时必须带 call_id
{
  "type": "function_call_output",
  "call_id": "call_abc123",
  "output": "北京今天 25°C"
}
```

一次对话中可能并行调用多个函数，`call_id` 用于区分结果属于哪个调用。

---

## 八、与 Assistants API 的关系

| 时间 | 事件 |
|------|------|
| 2025.8.26 | Assistants API 标记为弃用（deprecated） |
| 2026.8.26 | Assistants API 彻底下线（sunset） |

Assistants API 的概念对应关系：

| Assistants API（旧） | Responses API（新） |
|---|---|
| Assistant 对象 | 不需要预创建，请求时直接传参 |
| Thread 对象 | Conversation 对象 |
| Message 对象 | Conversation Items |
| Run 对象 | 直接调用 `responses.create()` |

---

## 九、三者对比总结

| | Chat Completions | Assistants API | Responses API |
|---|---|---|---|
| **对话历史** | 自己存，每次手动拼 | 服务端自动存（Thread） | 三种方式可选 |
| **工具调用** | 手动循环：调用→执行→拼接→再调用 | Run 自动处理调用链 | 自动处理 + 原生工具 |
| **文件/知识库** | 自己实现 RAG | 内置 file_search | 内置 file_search |
| **代码执行** | 自己搭沙箱 | 内置 code_interpreter | 内置 |
| **请求模式** | 同步，一次调用一次返回 | 异步，需轮询 Run 状态 | 同步或流式 |
| **复杂度** | 低，但功能有限 | 高，4 个对象多步流程 | 中等，兼顾简洁和功能 |
| **状态** | 仍支持，但推荐迁移 | 2026.8.26 下线 | **未来方向** |

---

## 十、常见迁移错误

1. 还在读 `choices[0].message.content` → 应该用 `response.output_text`
2. 把所有 output 当 message → 推理、工具调用是不同 Item 类型
3. 手动传上下文时丢了 reasoning / function_call Items
4. 函数结果没带 `call_id`
5. 还在用 `response_format` → 应该用 `text.format`
6. 直接复用旧的流式 handler → 需要按事件类型分支
7. 以为 `previous_response_id` 不收费 → **历史 token 仍然计费**

---

## 十一、迁移建议

Chat Completions 仍然支持，可以逐步迁移：

```
1. 从简单的文本生成流程开始
2. 改端点、请求体、输出处理
3. 选择上下文管理方式（previous_response_id / Items / Conversations）
4. 迁移函数定义，确认 call_id 正确
5. 结构化输出从 response_format 改到 text.format
6. 流式处理改成按事件类型分发
7. 能用原生工具的地方替换自定义实现
8. 对比行为、延迟、token 用量，确认后再迁移更多流量
```

---

## 十二、Output Item 类型详解

Responses API 的 `output` 不再只是消息，而是包含多种类型的 Item 列表：

| Item 类型 | 说明 |
|-----------|------|
| `message` | 普通文本消息（assistant 回复） |
| `reasoning` | 模型的推理过程（o1/o3 等推理模型才有） |
| `function_call` | 模型发起的函数调用请求 |
| `function_call_output` | 你提交的函数调用结果 |
| `web_search_call` | 内置搜索工具的调用记录 |
| `file_search_call` | 内置文件检索工具的调用记录 |
| `computer_call` | 内置电脑操控工具的调用记录 |

```python
response = client.responses.create(model="gpt-4o", input="你好")

for item in response.output:
    print(item.type)  # "message"
    print(item.content[0].text)  # "你好！有什么可以帮你的吗？"
```

### 获取推理内容

推理模型的推理过程默认不返回，需要用 `include` 参数显式请求：

```python
response = client.responses.create(
    model="o3-mini",
    input="证明根号2是无理数",
    include=["reasoning.encrypted_content"]  # 包含加密的推理内容
)

for item in response.output:
    if item.type == "reasoning":
        print(item.encrypted_content)  # 加密的推理摘要
```

---

## 十三、instructions 与 system message 的区别

### Chat Completions — 通过 system 角色设定

```python
messages = [
    {"role": "system", "content": "你是一个严谨的数学老师"},  # system 消息
    {"role": "user", "content": "1+1等于几？"}
]
```

system 消息是 messages 列表中的一条消息，和历史消息混在一起。

### Responses API — 独立的 instructions 参数

```python
response = client.responses.create(
    model="gpt-4o",
    instructions="你是一个严谨的数学老师",  # 独立的顶层参数
    input="1+1等于几？"
)
```

**关键区别**：`instructions` 不参与对话历史的传递。使用 `previous_response_id` 时，历史上下文会带过来，但 `instructions` **不会**从上一次响应中继承，需要每次都重新传。

---

## 十四、store 参数与 ZDR（零数据保留）

Responses API 默认会存储对话数据（`store: true`），用于支持 `previous_response_id` 等功能。

对于隐私合规场景（医疗、金融等），可以设置为不存储：

```python
response = client.responses.create(
    model="gpt-4o",
    input="用户的敏感问题",
    store=False  # 不在服务端存储对话数据
)
```

### ZDR 场景下的上下文传递

`store: false` 意味着服务端不存任何东西，包括推理内容。如果用的是推理模型（o1/o3）且需要跨轮次保持推理上下文，必须手动传回加密的推理 Item：

```python
# 第一轮
res1 = client.responses.create(
    model="o3-mini",
    input="问题1",
    store=False,
    include=["reasoning.encrypted_content"]
)

# 找到推理 Item
reasoning_item = None
for item in res1.output:
    if item.type == "reasoning":
        reasoning_item = item
        break

# 第二轮 — 手动带上推理内容
res2 = client.responses.create(
    model="o3-mini",
    input=[
        reasoning_item,  # 手动传回加密的推理内容
        {"role": "user", "content": "问题2"}
    ],
    store=False
)
```

---

## 十五、多模态输入（图片）对比

### Chat Completions

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "这张图片里有什么？"},
            {"type": "image_url", "image_url": {"url": "https://example.com/img.png"}}
        ]
    }]
)
```

### Responses API

```python
response = client.responses.create(
    model="gpt-4o",
    input=[
        {"role": "user", "content": "这张图片里有什么？"},
        {"role": "user", "content": [
            {"type": "input_image", "image_url": "https://example.com/img.png"}
        ]}
    ]
)
```

注意类型名变了：`image_url` → `input_image`。

---

## 十六、完整实战对比：带函数调用的多轮对话

### Chat Completions 版本

```python
from openai import OpenAI
import json

client = OpenAI()

tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "获取天气",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string"}
            },
            "required": ["city"]
        }
    }
}]

messages = [{"role": "system", "content": "你是一个有用的助手"}]

def chat(user_msg):
    global messages
    messages.append({"role": "user", "content": user_msg})
    
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        tools=tools
    )
    
    msg = response.choices[0].message
    messages.append(msg)
    
    # 处理函数调用
    if msg.tool_calls:
        for tool_call in msg.tool_calls:
            result = execute_tool(tool_call.function.name, tool_call.function.arguments)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result
            })
        
        # 需要再调一次让模型生成回复
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools
        )
        return response.choices[0].message.content
    
    return msg.content

def execute_tool(name, args):
    if name == "get_weather":
        city = json.loads(args)["city"]
        return f"{city}今天25°C，晴天"
    return "未知工具"

print(chat("北京天气怎么样？"))
```

### Responses API 版本

```python
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-4o",
    instructions="你是一个有用的助手",
    tools=[{
        "type": "function",
        "name": "get_weather",
        "description": "获取天气",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string"}
            },
            "required": ["city"]
        }
    }],
    input="北京天气怎么样？"
)

# 检查是否有函数调用
for item in response.output:
    if item.type == "function_call":
        print(f"调用函数: {item.name}")
        print(f"参数: {item.arguments}")
        
        # 执行函数，提交结果
        result = f"{item.arguments}今天25°C，晴天"
        
        response = client.responses.create(
            model="gpt-4o",
            instructions="你是一个有用的助手",
            tools=[{...}],  # 同上
            input=[
                {
                    "type": "function_call_output",
                    "call_id": item.call_id,
                    "output": result
                }
            ],
            previous_response_id=response.id
        )
        print(response.output_text)
    elif item.type == "message":
        print(item.content[0].text)
```

**核心区别**：
- Chat Completions 需要自己维护 messages 列表、自己处理循环调用
- Responses API 通过 `previous_response_id` 自动管理上下文，不需要手动拼接

---

## 十七、实用建议

### 什么时候用哪个上下文管理方式

| 场景 | 推荐方式 |
|------|---------|
| 简单的多轮聊天 | `previous_response_id` |
| 需要控制 token 用量，裁剪历史 | 手动传 Items |
| 长对话、跨会话持久化 | Conversations API |
| 隐私合规、零数据保留 | `store: false` + 手动传 Items |
| 应用重启后恢复对话 | Conversations API 或手动传 Items |

### `+=` 拼接历史的小技巧

```python
# 传统写法
messages = messages + [{"role": "user", "content": "新问题"}]  # 创建新列表

# 更高效 — 原地修改，不创建新对象
messages += [{"role": "user", "content": "新问题"}]  # 等价于 extend()
```

### 迁移优先级建议

```
高优先级（收益大、改动小）：
  ├── 新的文本生成流程 → 直接用 Responses API
  └── 新项目 → 直接基于 Responses API 开发

中优先级（收益中等、需要适配）：
  ├── 有函数调用的流程 → 改工具定义结构 + 处理 call_id
  └── 有结构化输出的流程 → response_format 改 text.format

低优先级（改动大、不急）：
  ├── 复杂的流式处理 → 需要重写事件处理逻辑
  └── Assistants API 用户 → 有一年过渡期，逐步迁移
```
