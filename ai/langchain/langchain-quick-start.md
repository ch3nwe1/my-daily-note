# LangChain 快速入门

**日期**：2026-08-16  
**主题**：LangChain 框架入门学习

## 目录

1. [LangChain 简介](#1-langchain-简介)
2. [DeepSeek 安装与使用](#2-deepseek-安装与使用)
3. [langchain-openai vs langchain-deepseek](#3-langchain-openai-vs-langchain-deepseek)
4. [API 兼容性设计](#4-api-兼容性设计)
5. [init_chat_model 与 ChatOpenAI](#5-init_chat_model-与-chatopenai)
6. [Checkpointer 机制](#6-checkpointer-机制)
7. [LangChain Agents vs Deep Agents](#7-langchain-agents-vs-deep-agents)
8. [构建真实世界的 Agent](#8-构建真实世界的-agent)
9. [常见问题与解决方案](#9-常见问题与解决方案)

---

## 1. LangChain 简介

### 什么是 LangChain

LangChain 是一个用于构建 LLM 应用的框架，核心思想是将大模型的能力与各种外部工具、数据源连接起来，形成完整的应用。

### 核心组件

- **Models**：模型接口（ChatOpenAI、ChatDeepSeek 等）
- **Prompts**：提示词模板
- **Chains**：链式调用
- **Memory**：记忆系统
- **Agents**：智能体
- **Tools**：工具集

### 官方文档

- Python 版：https://python.langchain.com/
- 官网：https://www.langchain.com/

---

## 2. DeepSeek 安装与使用

### 获取 API Key

1. 访问 [DeepSeek 开放平台](https://platform.deepseek.com/)
2. 注册账号并登录
3. 在控制台中创建 API Key

### 安装依赖

```bash
# 方式1：使用 langchain-openai（推荐，行业标准）
pip install langchain-openai

# 方式2：使用 langchain-deepseek（官方专用包）
pip install langchain-deepseek
```

### 基础使用示例

#### 使用 langchain-openai

```python
from langchain_openai import ChatOpenAI
import os

# 设置环境变量：export DEEPSEEK_API_KEY="你的API_KEY"

model = ChatOpenAI(
    model="deepseek-chat",  # 或 "deepseek-reasoner"
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com"  # 关键：指向 DeepSeek
)

response = model.invoke("你好，请用一句话介绍你自己")
print(response.content)
```

#### 使用 langchain-deepseek

```python
from langchain_deepseek import ChatDeepSeek
import os

model = ChatDeepSeek(
    model="deepseek-chat",
    api_key=os.getenv("DEEPSEEK_API_KEY")
)

response = model.invoke("你好")
print(response.content)
```

### 可用模型

- `deepseek-chat`：通用对话模型（便宜）
- `deepseek-reasoner`：推理模型，适合数学、逻辑题（稍贵）
- `deepseek-v4-flash`：较新的模型

---

## 3. langchain-openai vs langchain-deepseek

### 对比表

| 特性 | langchain-openai | langchain-deepseek |
|------|-----------------|-------------------|
| **通用性** | ✅ 支持多种兼容 API | ❌ 只支持 DeepSeek |
| **行业标准** | ✅ OpenAI 格式 | ❌ DeepSeek 特有格式 |
| **推理过程** | ❌ 丢失 reasoning_content | ✅ 完整支持 |
| **依赖数量** | ✅ 更少 | ❌ 更多 |
| **代码简洁度** | 需要设置 base_url | ✅ 不需要 |

### 选择建议

**选择 langchain-openai 当你**：
- ✅ 希望遵循行业标准
- ✅ 可能切换到其他 OpenAI 兼容 API
- ✅ 不需要 DeepSeek 特有的推理过程（reasoning_content）
- ✅ 想减少依赖

**选择 langchain-deepseek 当你**：
- ✅ 只使用 DeepSeek
- ✅ 需要完整的推理过程（reasoning_content）
- ✅ 想要最简洁的代码

### 决策

**本次学习选择 langchain-openai**，原因：
1. OpenAI API 是行业标准
2. 很多模型都兼容 OpenAI 格式
3. 切换模型成本低
4. 减少依赖

---

## 4. API 兼容性设计

### LangChain 的分层架构

```
┌─────────────────────────────────────┐
│      应用代码                        │  ← 用户代码
├─────────────────────────────────────┤
│   BaseChatModel (抽象接口)           │  ← 统一接口
├──────────┬──────────┬───────────────┤
│ ChatOpenAI│ChatDeepSeek│ChatAnthropic│ ← 具体实现
├──────────┴──────────┴───────────────┤
│   OpenAI API │ DeepSeek API │ Claude API │ ← 实际服务
└─────────────────────────────────────┘
```

### 耦合关系

| 层级 | 是否耦合 | 说明 |
|------|----------|------|
| **应用代码** | ❌ 不耦合 | 使用 `llm.invoke()`，不关心具体模型 |
| **接口层** | ❌ 不耦合 | `BaseChatModel` 定义统一接口 |
| **实现层** | ✅ 耦合 | `ChatDeepSeek` 只用于 DeepSeek |
| **API 层** | ✅ 耦合 | 每个模型有自己的 API |

### 好处

- 可以在不改动业务代码的情况下切换模型
- 学习一个接口，就能用所有模型
- 方便做 A/B 测试、模型对比

### 切换模型示例

```python
# 现在用 DeepSeek
from langchain_openai import ChatOpenAI
model = ChatOpenAI(
    model="deepseek-chat",
    api_key="DEEPSEEK_KEY",
    base_url="https://api.deepseek.com"
)

# 以后换成 OpenAI，只需要改这三行
model = ChatOpenAI(
    model="gpt-4",
    api_key="OPENAI_KEY"
)

# 下面的代码完全不用改
response = model.invoke("你好")
```

---

## 5. init_chat_model 与 ChatOpenAI

### 核心区别

| 特性 | init_chat_model | ChatOpenAI |
|------|----------------|------------|
| **类型** | 工厂函数 | 具体的类 |
| **作用** | 根据模型名自动选择正确的类 | 专门用于 OpenAI 兼容 API |
| **支持的模型** | 20+ 提供商的所有模型 | 任何 OpenAI 兼容的 API |
| **代码量** | 更少，更简洁 | 需要明确导入对应的类 |
| **灵活性** | 高，一行代码切换任意模型 | 低，只能调用 OpenAI 兼容 API |
| **控制力** | 较低，可能丢失某些特有参数 | 高，可以访问所有 OpenAI 参数 |

### 使用示例

#### init_chat_model

```python
from langchain.chat_models import init_chat_model

# 用 DeepSeek
model = init_chat_model(model="deepseek-chat", api_key="...")

# 换成 OpenAI，只改一行
model = init_chat_model(model="gpt-4", api_key="...")

# 换成 Claude
model = init_chat_model(model="claude-3-opus", api_key="...")
```

#### ChatOpenAI

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(
    model="deepseek-chat",
    api_key="...",
    base_url="https://api.deepseek.com",
    temperature=0.7,
    max_tokens=2000,
    # 可以访问所有 OpenAI 兼容的参数
)
```

### 选择建议

| 场景 | 推荐 |
|------|------|
| 快速实验，可能切换多个模型 | init_chat_model ✅ |
| 生产环境，需要精细控制 | ChatOpenAI ✅ |
| 学习阶段，想简化代码 | init_chat_model ✅ |
| 需要访问提供商特有的高级参数 | 具体的类 ✅ |

### 注意事项

使用 `init_chat_model` 调用 DeepSeek 时：
- ❌ `base_url` 参数不起作用
- ✅ 使用 `api_base` 替代

---

## 6. Checkpointer 机制

### 什么是 Checkpointer

**Checkpointer** = **检查点器**，用于保存和恢复 Agent 的状态。

类比：玩游戏时的存档系统
- 没有 checkpointer → 每次都要从头开始
- 有 checkpointer → 可以从存档点继续

### 为什么需要

**没有 Checkpointer**：
```python
# 第一次对话
agent.invoke({"messages": [{"role": "user", "content": "我叫小明"}]})
# Agent: "你好小明！"

# 第二次对话
agent.invoke({"messages": [{"role": "user", "content": "我叫什么名字？"}]})
# Agent: "抱歉，我不知道" ❌
```

**有 Checkpointer**：
```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()

agent = create_agent(
    model=model,
    checkpointer=checkpointer,
    thread_id="user_123"
)

# 第一次对话
agent.invoke(
    {"messages": [{"role": "user", "content": "我叫小明"}]},
    config={"configurable": {"thread_id": "user_123"}}
)

# 第二次对话
agent.invoke(
    {"messages": [{"role": "user", "content": "我叫什么名字？"}]},
    config={"configurable": {"thread_id": "user_123"}}
)
# Agent: "你叫小明啊！" ✅
```

### 核心功能

1. **保存状态**：每一步的状态都会被保存
2. **恢复状态**：可以从任意检查点恢复
3. **持久化**：可以存到内存、数据库、文件

### 不同类型

#### MemorySaver（内存存储）

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
# 优点：简单快速
# 缺点：程序重启后数据丢失
```

#### SqliteSaver（SQLite 数据库）

```python
from langgraph.checkpoint.sqlite import SqliteSaver

checkpointer = SqliteSaver.from_conn_string("checkpoints.sqlite")
# 优点：程序重启后数据还在
# 缺点：只适合单机使用
```

#### PostgresSaver（PostgreSQL 数据库）

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:password@localhost/dbname"
)
# 优点：适合生产环境，支持多用户
# 缺点：需要数据库
```

### 应用场景

1. **多轮对话记忆**：记住对话历史
2. **人工介入**：在执行敏感操作前暂停，等待审批
3. **错误恢复**：从最后一个检查点恢复，不用从头开始

---

## 7. LangChain Agents vs Deep Agents

### 两种框架对比

| 特性 | LangChain Agents | Deep Agents |
|------|-----------------|-------------|
| **控制程度** | 🎯 精细控制 | 🎯 中等控制 |
| **内置功能** | ❌ 基础 | ✅ 丰富（规划、文件工具、子Agent） |
| **配置复杂度** | 🔧 高（需要手动组装） | 🔧 低（开箱即用） |
| **学习曲线** | 📚 陡峭 | 📚 平缓 |
| **适用场景** | 需要高度定制的项目 | 快速构建功能强大的 Agent |
| **灵活性** | ✅ 非常高 | ✅ 高 |
| **代码量** | 多 | 少 |

### Deep Agents 的内置功能

#### 1. Planning（规划能力）

```python
# Deep Agent 可以自动分解复杂任务
用户: "帮我写一篇关于 AI 的研究报告"

Deep Agent 内部思考:
1. 先搜索 AI 的历史
2. 然后搜索 AI 的现状
3. 再搜索 AI 的未来趋势
4. 整合所有信息
5. 生成报告
```

#### 2. File System Tools（文件系统工具）

```python
# Deep Agent 内置了文件操作工具
agent.read_file("paper.pdf")
agent.write_file("report.md", content)
agent.list_files("./research/")
```

#### 3. Subagents（子 Agent）

```python
# Deep Agent 可以创建子 Agent 处理不同任务
主 Agent: 分配任务
  ├─ 子Agent1: 搜索历史信息
  ├─ 子Agent2: 分析当前数据
  └─ 子Agent3: 预测未来趋势
```

### 代码对比

#### LangChain Agents

```python
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI
from langgraph.checkpoint.memory import MemorySaver

# 需要自己定义所有组件
model = ChatOpenAI(model="deepseek-chat", ...)

# 需要自己定义工具
@tool
def search(query: str) -> str:
    """搜索信息"""
    return "搜索结果..."

# 需要自己配置记忆
checkpointer = MemorySaver()

# 需要自己组装 Agent
agent = create_agent(
    model=model,
    tools=[search],
    system_prompt="你是一个研究助手",
    checkpointer=checkpointer,
)
```

#### Deep Agents

```python
from deep_agents import DeepAgent

# 内置了很多功能
agent = DeepAgent(
    model="deepseek-chat",
    enable_planning=True,      # 内置的规划功能
    enable_file_tools=True,    # 内置的文件操作工具
    enable_subagents=True      # 内置的子 Agent 功能
)
```

### 选择建议

**选择 LangChain Agents 当你**：
- ✅ 需要完全控制每个细节
- ✅ 有特殊的需求，需要自定义逻辑
- ✅ 学习 LangChain 的底层机制
- ✅ 构建生产级的定制化应用

**选择 Deep Agents 当你**：
- ✅ 想快速构建功能强大的 Agent
- ✅ 不想手动实现规划、文件操作等常见功能
- ✅ 需要开箱即用的解决方案

### 类比

| 框架 | 类比 |
|------|------|
| **LangChain Agents** | 像买零件自己组装电脑 🖥️ |
| **Deep Agents** | 像买品牌笔记本电脑 💻 |

---

## 8. 构建真实世界的 Agent

### "Build a real-world agent" 的含义

构建一个**真正能用、有实际价值的 AI Agent**，而不是简单的 demo。

### 简单示例 vs 真实世界 Agent

| 简单示例 | 真实世界 Agent |
|---------|---------------|
| "今天天气怎么样？" → 返回固定文本 | 调用真实天气 API，获取实时数据 |
| 硬编码的工具函数 | 连接数据库、API、文件系统 |
| 单轮对话 | 多轮对话，有记忆 |
| 没有错误处理 | 处理异常、重试、回退 |
| 本地运行 | 部署到生产环境 |
| 没有日志/监控 | 有追踪、可观测性 |

### 真实世界 Agent 的特点

1. **连接真实数据源**
   - 调用外部 API（天气、股票、搜索）
   - 访问数据库
   - 读取文件系统

2. **处理复杂任务**
   - 多步骤推理
   - 使用多个工具
   - 处理不确定性

3. **有记忆和状态**
   - 记住对话历史
   - 跨会话保持上下文
   - 持久化存储

4. **生产级特性**
   - 错误处理和重试
   - 日志和追踪
   - 性能优化
   - 安全控制（护栏、审批）

### 教程中的 6 个核心概念

1. **Detailed system prompts**（详细的系统提示词）
   - 给 Agent 写的"行为指南"
   - 让 Agent 的行为更可控、更可预测

2. **Create tools that integrate with external data**（创建与外部数据集成的工具）
   - 让 Agent 能访问外部数据
   - 连接真实世界的信息

3. **Model configuration for consistent responses**（模型配置以获得一致的响应）
   - 设置 temperature、max_tokens 等参数
   - 让输出更稳定

4. **Conversational memory for chat-like interactions**（对话记忆）
   - 让 Agent 记住之前的对话
   - 实现多轮交互

5. **Deep Agents for built-in features**（使用 Deep Agents 获取内置功能）
   - 规划、文件工具、子 Agent
   - 减少手动配置

6. **Testing your agent**（测试你的 Agent）
   - 单元测试、集成测试、边界测试
   - 确保可靠性

---

## 9. 常见问题与解决方案

### 问题1：SSL 证书错误

**错误信息**：
```
SSL: CERTIFICATE_VERIFY_FAILED
certificate verify failed: self-signed certificate in certificate chain
```

**原因**：
- 网络环境有 SSL 拦截（公司内网、代理服务器）
- 代理使用自签名证书，不被系统信任

**解决方案**：

#### 方案1：禁用 SSL 验证（临时）

```python
import ssl
import urllib.request

ssl_context = ssl.create_default_context()
ssl_context.check_hostname = False
ssl_context.verify_mode = ssl.CERT_NONE

response = urllib.request.urlopen(url, context=ssl_context)
```

#### 方案2：使用本地文件

```python
# 手动下载文件到本地，然后读取
@tool
def read_local_file(file_path: str) -> str:
    """读取本地文件"""
    with open(file_path, 'r', encoding='utf-8') as f:
        return f.read()
```

#### 方案3：使用 HTTP 而不是 HTTPS

```python
url = "http://www.gutenberg.org/files/64317/64317-0.txt"  # HTTP
```

### 问题2：非标准响应字段丢失

**警告信息**：
```
Non-standard response fields added by third-party providers 
(e.g., `reasoning_content`, `reasoning_details`) are **not** 
extracted or preserved.
```

**含义**：
- 使用 `ChatOpenAI` 调用 DeepSeek 时，会丢失 `reasoning_content` 等特有字段
- 如果需要完整的推理过程，应该使用 `ChatDeepSeek`

**解决方案**：
- 不需要推理过程 → 可以忽略警告
- 需要推理过程 → 安装 `langchain-deepseek`，使用 `ChatDeepSeek`

### 问题3：create_agent 的参数

**正确用法**：

```python
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI

model = ChatOpenAI(
    model="deepseek-chat",
    api_key="你的API_KEY",
    base_url="https://api.deepseek.com"
)

agent = create_agent(
    model=model,  # 传入模型实例
    tools=[工具函数列表],
    system_prompt="系统提示词",
    checkpointer=MemorySaver()  # 可选
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "你的问题"}]}
)
```

---

## 学习路径建议

```
1. 安装配置 → 2. 基础调用 → 3. 理解架构 → 4. 构建工具 → 5. 添加记忆 → 6. 测试优化
     ↓            ↓           ↓            ↓            ↓            ↓
  环境准备     简单示例    分层设计      工具开发    Checkpointer   生产部署
```

### 推荐步骤

1. **先用 langchain-openai 学习基础**
   - 理解统一接口设计
   - 掌握模型切换方法

2. **学习核心概念**
   - Tools、Chains、Memory
   - Agents 的工作原理

3. **构建真实世界的 Agent**
   - 连接真实 API
   - 添加记忆系统
   - 实现错误处理

4. **根据需求选择框架**
   - 需要精细控制 → LangChain Agents
   - 需要快速开发 → Deep Agents

---

## 关键决策记录

### 决策1：使用 langchain-openai 而不是 langchain-deepseek

**原因**：
- OpenAI API 是行业标准
- 切换模型成本低
- 减少依赖
- 更通用

**代价**：
- 无法获取 DeepSeek 的 reasoning_content

### 决策2：学习阶段优先使用具体类

**原因**：
- 更清晰，便于理解底层原理
- 控制力强，适合生产环境
- 遇到问题更容易调试

---

## 下一步学习

1. ✅ 完成研究 Agent 的构建
2. 🔲 深入学习 Prompt 工程
3. 🔲 掌握 Memory 系统
4. 🔲 学习 RAG（检索增强生成）
5. 🔲 探索多 Agent 协作
6. 🔲 部署到生产环境

---

## 参考资源

### 官方文档

- LangChain Python: https://python.langchain.com/
- LangChain GitHub: https://github.com/langchain-ai/langchain
- DeepSeek 平台: https://platform.deepseek.com/

### 相关笔记

- [OpenAI Agents SDK 学习笔记](../openai/)（在同级目录）

---

**总结**：本次会话系统学习了 LangChain 框架的基础知识，包括安装配置、核心概念、架构设计、常见问题解决等。明确了使用 langchain-openai 作为主要学习路径，为后续深入学习 Agent 开发打下基础。
