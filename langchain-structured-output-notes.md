# LangChain 结构化输出学习笔记

## 一、结构化输出概述

LangChain 的 `create_agent` 支持自动处理结构化输出。用户设置期望的 schema，模型生成结构化数据时会被捕获、校验，并存储在 `result['structured_response']` 中。

### 响应格式策略

使用 `response_format` 控制如何返回结构化数据：

- **`ProviderStrategy[SchemaT]`**：使用提供商原生结构化输出（OpenAI、Anthropic、xAI 等）
- **`ToolStrategy[SchemaT]`**：使用工具调用实现结构化输出
- **`type[SchemaT]`**：直接传 schema 类型，LangChain 自动选择最佳策略
- **`None`**：不使用结构化输出

当直接传 schema 类型时，LangChain 自动选择：
- 模型支持原生结构化输出 → `ProviderStrategy`
- 否则 → `ToolStrategy`

---

## 二、ProviderStrategy（原生结构化输出）

### 基本用法

```python
from langchain.agents.structured_output import ProviderStrategy

class UserResponse(BaseModel):
    name: str
    age: int

agent = create_agent(
    model="openai:gpt-4o",
    tools=[...],
    response_format=ProviderStrategy(
        schema=UserResponse,
        strict=True  # 需要 langchain>=1.2
    )
)
```

### strict 参数

- `strict=True`：严格遵循 schema，所有字段必须出现
- `strict=False` 或 `None`（默认）：灵活输出
- 仅部分提供商支持（OpenAI、xAI 等）

### 支持的 schema 类型

- **Pydantic 模型**：返回校验后的实例
- **Dataclasses**：返回 dict
- **TypedDict**：返回 dict
- **JSON Schema**：返回 dict（必须包含 `title` 和 `description`）

---

## 三、ToolStrategy（工具调用方式）

### 基本原理

对于不支持原生结构化输出的模型，LangChain 把 schema 转换成工具定义：

```python
# Pydantic 模型
class MeetingAction(BaseModel):
    task: str
    assignee: str
    priority: Literal["low", "medium", "high"]

# 转换成工具定义
{
  "type": "function",
  "function": {
    "name": "MeetingAction",
    "parameters": {
      "type": "object",
      "properties": {
        "task": {"type": "string"},
        "assignee": {"type": "string"},
        "priority": {"type": "string", "enum": ["low", "medium", "high"]}
      },
      "required": ["task", "assignee", "priority"]
    }
  }
}
```

然后设置 `tool_choice="required"` 强制模型调用该工具。

### 重要限制

**Thinking 模式与工具调用不兼容：**

- 所有支持 thinking 模式的模型（o1、o3-mini、Claude、Qwen with thinking）都不支持 `tool_choice="required"`
- 原因：thinking 模型需要自主决定是否调用工具，强制调用会破坏推理过程
- 解决方案：使用工具时禁用 thinking 模式

### ToolStrategy 参数

```python
class ToolStrategy(Generic[SchemaT]):
    schema: type[SchemaT]
    tool_message_content: str | None  # 自定义工具消息内容
    handle_errors: Union[bool, str, type[Exception], ...]  # 错误处理策略
```

**tool_message_content：**
- 自定义工具消息内容（在结构化数据之前）
- 不影响结构化数据提取
- 用于影响 agent 后续行为

**handle_errors：**
- `True`（默认）：捕获错误，提示重试
- `False`：直接抛出异常，不重试
- `str`：用自定义消息提示重试
- `type[Exception]`：只捕获特定异常
- `Callable[[Exception], str]`：自定义错误处理函数

### 重试控制

重试次数由 `recursion_limit` 控制：

```python
agent = create_agent(
    model="gpt-4o",
    tools=[],
    response_format=ToolStrategy(schema=MeetingAction),
    recursion_limit=3  # 最多 3 次循环
)
```

---

## 四、Union 类型

### 用法

```python
from typing import Union

class ContactInfo(BaseModel):
    name: str
    email: str

class EventDetails(BaseModel):
    event_name: str
    date: str

agent = create_agent(
    model="gpt-4o",
    tools=[],
    response_format=ToolStrategy(Union[ContactInfo, EventDetails])
)
```

### JSON Schema 表示

Union 类型转换成 `anyOf`：

```json
{
  "anyOf": [
    {"$ref": "#/definitions/ContactInfo"},
    {"$ref": "#/definitions/EventDetails"}
  ]
}
```

LangChain 会把每个模型转换成独立的工具，模型选择调用其中一个。

### 错误处理

如果模型错误地调用了多个工具：

```
第 1 次：模型调用 ContactInfo 和 EventDetails
    ↓
Agent 检测到错误：调用了多个结构化输出工具
    ↓
返回错误消息："Error: Model incorrectly returned multiple structured responses..."
    ↓
第 2 次：模型只调用一个工具
    ↓
成功返回结构化数据
```

---

## 五、Python 泛型基础

### TypeVar

```python
from typing import TypeVar

# 定义类型变量
SchemaT = TypeVar("SchemaT")

# 带约束
SchemaT = TypeVar("SchemaT", bound=BaseModel)  # 必须是 BaseModel 子类
```

### Generic

```python
from typing import TypeVar, Generic

T = TypeVar("T")

class Container(Generic[T]):
    def __init__(self, value: T):
        self.value = value

# 使用
c: Container[str] = Container("hello")
# IDE 知道 c.value 是 str 类型
```

### 继承 + 泛型

```python
class Child(Parent, Generic[T]):  # 父类在前，Generic 在后
    def __init__(self, value: T):
        super().__init__()
        self.value = value
```

### Union 语法

```python
# Python 3.7+
from typing import Union
x: Union[int, str]

# Python 3.10+
x: int | str  # 语法糖，效果相同
```

LangChain 使用 `Union` 是为了兼容旧版本 Python。

---

## 六、TypedDict 使用

### 基本用法

```python
from typing import TypedDict

class UserResponse(TypedDict):
    name: str
    age: int

# 返回 dict
result: UserResponse = {"name": "John", "age": 30}
```

### 添加字段描述

```python
from typing import TypedDict, Annotated

class ContractInfo(TypedDict):
    name: Annotated[str, "The name of the person."]
    email: Annotated[str, "The email address of the person."]
```

### TypedDict vs Pydantic

| 特性 | TypedDict | Pydantic |
|------|-----------|----------|
| 返回值 | dict | 对象实例 |
| 字段校验 | ❌ 无 | ✅ 有 |
| 依赖 | 标准库 | 需要安装 |
| 推荐场景 | 简单结构 | 复杂场景 |

---

## 七、常见问题

### 1. thinking 模式报错

```
The tool_choice parameter does not support being set to required or object in thinking mode
```

**原因：** thinking 模式不支持 `tool_choice="required"`

**解决：** 禁用 thinking 模式

```python
llm = ChatOpenAI(
    model="qwen3.8-max",
    extra_body={"enable_thinking": False}
)
```

### 2. 字符串 "None" 校验错误

```
rating: Input should be a valid integer, unable to parse string as an integer
[type=int_parsing, input_value='None', input_type=str]
```

**原因：** 模型返回字符串 `"None"` 而不是真正的 `null/None`

**解决方案：**

```python
# 方案 1：字段校验器（推荐）
class Review(BaseModel):
    rating: Optional[int]
    
    @field_validator('rating', mode='before')
    @classmethod
    def clean_rating(cls, v):
        if isinstance(v, str) and v.lower() in ["none", "null"]:
            return None
        return v

# 方案 2：改进描述
rating: Optional[int] = Field(
    description="Use null for no rating, NEVER use string 'None'"
)

# 方案 3：handle_errors 重试
response_format=ToolStrategy(
    schema=Review,
    handle_errors=lambda e: "请使用整数或 null，不要返回字符串 'None'"
)
```

### 3. PyCharm 类型警告

`handle_errors` 传字符串或函数时 PyCharm 可能警告，但代码能正常运行。这是 PyCharm 类型检查器的限制，不是代码错误。

**解决：**
- 忽略警告
- 使用 `# noinspection PyTypeChecker`
- 只用 `bool` 类型

---

## 八、最佳实践

### 1. 选择策略

```python
# 支持原生结构化输出的模型（GPT-4o、Claude 等）
response_format=ProviderStrategy(schema=MySchema, strict=True)

# 其他模型
response_format=ToolStrategy(schema=MySchema)

# 不确定时，直接传类型
response_format=MySchema  # LangChain 自动选择
```

### 2. 使用工具时

```python
# 禁用 thinking 模式
llm = ChatOpenAI(model="...", extra_body={"enable_thinking": False})

# 设置重试限制
agent = create_agent(
    model=llm,
    tools=[...],
    response_format=ToolStrategy(schema=MySchema),
    recursion_limit=3
)
```

### 3. 处理模型错误输出

```python
# 组合使用多种方法
class MySchema(BaseModel):
    model_config = ConfigDict(strict=True)
    
    field: Optional[int] = Field(
        description="Use integer or null, NEVER string 'None'",
        examples=[5, 3, None]
    )
    
    @field_validator('field', mode='before')
    @classmethod
    def clean_field(cls, v):
        if isinstance(v, str) and v.lower() in ["none", "null"]:
            return None
        return v

# 配合错误处理
response_format=ToolStrategy(
    schema=MySchema,
    handle_errors=lambda e: f"请修正: {str(e)}"
)
```

---

## 九、总结

| 概念 | 关键点 |
|------|--------|
| ProviderStrategy | 原生支持，最可靠，需要模型支持 |
| ToolStrategy | 转成工具调用，兼容性好 |
| strict | 强制 schema 遵循，部分模型支持 |
| thinking 模式 | 与工具调用不兼容，用工具时禁用 |
| Union 类型 | 多个 schema 选择，模型选一个 |
| handle_errors | 控制错误处理，不控制重试次数 |
| recursion_limit | 控制最大重试次数 |
| 泛型语法 | `Union` 兼容性好，`\|` 更简洁（3.10+） |

**核心原则：**
1. 优先使用 ProviderStrategy（如果模型支持）
2. 使用工具时禁用 thinking 模式
3. 用 field_validator 防御性处理模型错误
4. 组合使用多种方法提高可靠性
