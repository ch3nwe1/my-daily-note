# OpenAI Agents SDK 沙箱完整指南

## 目录

1. [沙箱概述](#一沙箱概述)
2. [架构设计](#二架构设计)
3. [何时使用沙箱](#三何时使用沙箱)
4. [核心组件](#四核心组件)
5. [Manifest 详解](#五manifest-详解)
6. [Capabilities 详解](#六capabilities-详解)
7. [运行沙箱代理](#七运行沙箱代理)
8. [状态管理](#八状态管理)
9. [沙箱记忆](#九沙箱记忆)
10. [工作流模式](#十工作流模式)
11. [最佳实践](#十一最佳实践)
12. [完整示例](#十二完整示例)

---

## 一、沙箱概述

### 什么是沙箱？

沙箱为代理提供一个**隔离的、类 Unix 的执行环境**，包括：

- **文件系统**：读写文件
- **Shell**：执行命令
- **已安装的包**：依赖管理
- **挂载的数据**：外部存储
- **暴露的端口**：运行服务
- **快照**：状态保存
- **受控的外部访问**：安全的网络访问

### 为什么需要沙箱？

```
普通代理：
用户提问 → 模型推理 → 返回文本
（只能在"脑子里"思考）

沙箱代理：
用户提问 → 模型推理 → 在沙箱中执行 → 返回结果
（能真正"动手做事"）
```

**核心价值**：
- ✅ 从"说"到"做"：代理不再只能建议，能真正执行
- ✅ 安全隔离：敏感操作在沙箱中，不影响主系统
- ✅ 状态持久化：工作可以暂停、恢复，不丢失进度
- ✅ 灵活扩展：从本地到 Docker 到云，平滑过渡

---

## 二、架构设计

### Harness vs Compute

这是沙箱的核心架构思想：

```
┌─────────────────────────────────────────────────────────┐
│  Harness（控制框架）- 运行在你的服务器                    │
│                                                         │
│  职责：                                                  │
│  ├─ 管理代理循环                                        │
│  ├─ 调用模型 API                                        │
│  ├─ 路由工具调用                                        │
│  ├─ 处理交接（handoffs）                                │
│  ├─ 审批流程                                            │
│  ├─ 追踪日志                                            │
│  ├─ 状态恢复                                            │
│  └─ 认证计费                                            │
│                                                         │
│  运行位置：你的可信服务器                                │
└─────────────────────────────────────────────────────────┘
                          │
                          │ API 调用
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Compute（计算环境）- 运行在沙箱提供商                    │
│                                                         │
│  职责：                                                  │
│  ├─ 读写文件                                            │
│  ├─ 执行命令                                            │
│  ├─ 安装依赖                                            │
│  ├─ 使用挂载存储                                        │
│  ├─ 暴露端口                                            │
│  └─ 快照状态                                            │
│                                                         │
│  运行位置：隔离的沙箱容器                                │
└─────────────────────────────────────────────────────────┘
```

**为什么要分开？**

- **安全性**：敏感信息（API 密钥）在 harness，沙箱访问不到
- **可移植性**：换个沙箱提供商（E2B → Docker），harness 不变
- **可恢复性**：沙箱崩溃了，harness 有状态快照，可以重启
- **审计性**：所有操作日志在 harness，不会因沙箱销毁而丢失

### 沙箱的实现方式

#### 1. 容器（Docker）

```
宿主机（你的电脑）
├─ 内核（共享）
├─ 容器 A（Python 沙箱）
│  ├─ 独立的文件系统
│  ├─ 独立的进程空间
│  ├─ 独立的网络栈
│  └─ 资源限制（CPU、内存）
└─ 容器 B（Node.js 沙箱）
   └─ ...
```

**特点**：
- ✅ 启动快（秒级）
- ✅ 资源占用小（MB 级）
- ✅ 共享宿主机内核
- ❌ 隔离性中等（内核漏洞可能逃逸）

#### 2. 虚拟机（VMs）

```
宿主机
├─ Hypervisor（虚拟机管理器）
├─ VM 1（完整的操作系统）
│  ├─ Guest OS（客户操作系统）
│  ├─ 独立的内核
│  ├─ 虚拟 CPU、内存、磁盘
│  └─ 虚拟网络
└─ VM 2
   └─ ...
```

**特点**：
- ✅ 隔离性强（硬件级隔离）
- ❌ 启动慢（分钟级）
- ❌ 资源占用大（GB 级）

#### 3. UnixLocal（本地执行）

```python
from agents.sandbox.sandboxes.unix_local import UnixLocalSandboxClient

client = UnixLocalSandboxClient()
```

**特点**：
- ✅ 最快（无容器开销）
- ✅ 容易调试
- ❌ 隔离性弱（只是路径限制）
- ❌ 不适合生产

**适用场景**：本地开发、快速原型、可信代码

---

## 三、何时使用沙箱

### 需要用沙箱的信号

- ✅ 任务需要操作整个文件目录，不是单个文件
- ✅ 代理需要生成文件供后续检查（报告、代码、数据）
- ✅ 需要运行命令、安装包、执行脚本
- ✅ 工作流产生工件（Markdown、CSV、截图、网站）
- ✅ 需要暴露端口运行服务（Jupyter、Web 服务器）
- ✅ 工作可暂停、人工审查、然后恢复

### 不需要沙箱的情况

- ❌ 只需要简短文本回答
- ❌ 不需要持久化工作空间
- ❌ 纯推理任务（问答、翻译、摘要）

### 决策流程

```
你的任务需要什么？
│
├─ 只需要推理提示词内容？
│  └─ 是 → 直接用 Responses API
│
├─ 需要运行命令，但只是偶尔？
│  └─ 是 → 用托管 Shell 工具
│
├─ 需要操作文件/生成工件/暴露端口/暂停恢复？
│  └─ 是 → 用沙箱代理
│
└─ 不确定？
   └─ 从托管 Shell 工具开始，需要时再升级到沙箱
```

---

## 四、核心组件

### 1. SandboxAgent

SandboxAgent 保持了通常的代理接口：

```python
agent = SandboxAgent(
    name="助手",
    instructions="你是一个有用的助手",
    model="gpt-4",
    tools=[...],
    handoffs=[...],
    mcp_servers=[...],
    output_type=MyOutput,
    guardrails=[...],
    hooks=[...],
    # 沙箱特定
    default_manifest=manifest,
    capabilities=[...],
)
```

**保持不变的部分**：
- instructions、prompt、tools
- handoffs、MCP servers
- model settings、structured output
- guardrails、hooks

**改变的部分**：
- 执行边界从"提示词内"变成"沙箱会话"

### 2. Manifest（清单）

Manifest 定义**新沙箱的初始状态**，是"契约"而非"完整状态"。

```python
from agents.sandbox import Manifest
from agents.sandbox.entries import File, Dir, GitRepo

manifest = Manifest(
    entries={
        "data/sales.csv": File(content=b"date,amount\n2024-01-01,1000"),
        "scripts/analyze.py": File(source="local://scripts/analyze.py"),
        "project": GitRepo(repo="https://github.com/user/project", ref="main"),
        "output": Dir(),
    },
    environment={
        "PYTHONPATH": "/workspace",
    }
)
```

**关键约束**：
- ✅ 路径必须是相对路径：`data/file.txt`
- ❌ 不能用绝对路径：`/Users/chenwei/file.txt`
- ❌ 不能路径逃逸：`../secrets/keys.txt`

### 3. Capabilities（能力）

Capabilities 给代理添加特定功能：

```python
from agents.sandbox.capabilities import (
    Shell,        # 执行命令
    Filesystem,   # 文件操作
    Memory,       # 跨运行记忆
    Skills,       # 可复用指令
    Compaction,   # 长上下文压缩
)

agent = SandboxAgent(
    capabilities=[
        Shell(),
        Filesystem(),
        Memory(),
    ]
)
```

**默认能力**：Shell、Filesystem、Compaction

### 4. Sandbox Client（沙箱客户端）

选择不同的沙箱提供商：

```python
# 本地开发
from agents.sandbox.sandboxes.unix_local import UnixLocalSandboxClient
client = UnixLocalSandboxClient()

# Docker 环境
from agents.sandbox.sandboxes.docker import DockerSandboxClient
client = DockerSandboxClient(docker_client)

# 托管服务（E2B、Modal 等）
# 使用对应的客户端
```

### 5. Sandbox Session（沙箱会话）

会话是活跃的执行环境：

```python
session = await client.create(manifest=manifest)

# 会话包含：
# - 文件系统状态
# - 环境变量
# - 运行中的进程
# - 提供商特定的元数据
```

### 6. Sandbox Run Config（运行配置）

每次运行的配置：

```python
run_config = RunConfig(
    sandbox=SandboxRunConfig(
        client=client,
        manifest=manifest,
        session=session,
        environment={...},
    )
)
```

---

## 五、Manifest 详解

### Manifest 是什么？

**Manifest = 沙箱的"蓝图"或"配方"**

它定义了：当创建一个新的沙箱时，里面应该有什么？

### Manifest 包含什么？

#### 1. 文件（File）

```python
from agents.sandbox.entries import File

manifest = Manifest(
    entries={
        "readme.md": File(
            content="# 项目说明\n\n这是示例项目".encode("utf-8")
        ),
        "config.json": File(
            content=b'{"debug": true}'
        ),
    }
)
```

#### 2. 目录（Dir）

```python
from agents.sandbox.entries import Dir

manifest = Manifest(
    entries={
        "output": Dir(),
        "temp": Dir(),
        "logs": Dir(),
    }
)
```

#### 3. 本地文件（LocalFile）

```python
from agents.sandbox.entries import LocalFile

manifest = Manifest(
    entries={
        "data/sales.csv": LocalFile(
            path="/Users/chenwei/data/sales_2024.csv"
        ),
    }
)
```

#### 4. Git 仓库（GitRepo）

```python
from agents.sandbox.entries import GitRepo

manifest = Manifest(
    entries={
        "project": GitRepo(
            repo="https://github.com/user/project",
            ref="main"
        ),
    }
)
```

#### 5. 云存储挂载

```python
from agents.sandbox.entries import S3Mount, GCSMount

manifest = Manifest(
    entries={
        "data/s3": S3Mount(bucket="my-bucket", prefix="datasets/"),
        "data/gcs": GCSMount(bucket="my-gcs-bucket"),
    }
)
```

#### 6. 环境变量

```python
manifest = Manifest(
    entries={...},
    environment={
        "DEBUG": "true",
        "API_BASE_URL": "https://api.example.com",
        "DATABASE_URL": "${DATABASE_URL}",  # 从环境变量获取
    }
)
```

### Manifest vs 实际工作空间

```
Manifest = "新沙箱应该长这样"（蓝图）

实际工作空间可能来自：
1. 根据 manifest 创建新沙箱
2. 复用之前的活跃沙箱
3. 从快照恢复
4. 从序列化的状态恢复
```

---

## 六、Capabilities 详解

### 1. Shell：执行命令

```python
from agents.sandbox.capabilities import Shell

agent = SandboxAgent(
    capabilities=[Shell()],
)
```

**添加的能力**：
- `shell_exec` 工具：执行命令
- 交互式输入支持
- 命令输出捕获

### 2. Filesystem：文件操作

```python
from agents.sandbox.capabilities import Filesystem

agent = SandboxAgent(
    capabilities=[Shell(), Filesystem()],
)
```

**添加的能力**：
- `apply_patch` 工具：应用补丁修改文件
- `view_image` 工具：查看图片文件
- 文件读写操作

### 3. Skills：技能系统

```python
from agents.sandbox.capabilities import Skills
from agents.sandbox.entries import GitRepo

agent = SandboxAgent(
    capabilities=[
        Skills(
            from_=GitRepo(
                repo="https://github.com/org/skills",
                ref="main"
            )
        ),
    ],
)
```

**三种技能源**：
- **Lazy Local Directory**：大型目录，按需加载
- **Local Directory**：小型目录，预加载
- **Git Repo**：版本控制，多沙箱共享

### 4. Memory：记忆系统

```python
from agents.sandbox.capabilities import Memory

agent = SandboxAgent(
    capabilities=[Shell(), Filesystem(), Memory()],
)
```

**记忆模式**：
- **默认读写**：读取现有记忆 + 生成新记忆
- **只读**：只读取，不生成
- **只生成**：只生成，不读取

### 5. Compaction：上下文压缩

```python
from agents.sandbox.capabilities import Compaction

agent = SandboxAgent(
    capabilities=[Shell(), Filesystem(), Compaction()],
)
```

**用途**：长时间运行的流程需要裁剪上下文

### 默认 Capabilities

```python
# 默认包含：Shell, Filesystem, Compaction

# 如果传 capabilities 列表，会替换默认列表
agent = SandboxAgent(
    capabilities=[
        Shell(),        # 手动包含
        Filesystem(),   # 手动包含
        Compaction(),   # 手动包含
        Memory(),       # 新增
    ]
)
```

---

## 七、运行沙箱代理

### 基本循环（5 步）

```python
import asyncio
from agents import Runner
from agents.run import RunConfig
from agents.sandbox import SandboxAgent, SandboxRunConfig, Manifest
from agents.sandbox.capabilities import Shell
from agents.sandbox.entries import File
from agents.sandbox.sandboxes.unix_local import UnixLocalSandboxClient


async def main():
    # 步骤 1：构建 Manifest
    manifest = Manifest(
        entries={
            "hello.py": File(content=b'print("Hello from Sandbox!")')
        }
    )
    
    # 步骤 2：创建 SandboxAgent
    agent = SandboxAgent(
        name="Shell 助手",
        instructions="使用 shell 命令完成任务。",
        capabilities=[Shell()],
    )
    
    # 步骤 3：选择 Sandbox Client
    client = UnixLocalSandboxClient()
    
    # 步骤 4：运行代理
    result = await Runner.run(
        agent,
        "运行 hello.py",
        run_config=RunConfig(
            sandbox=SandboxRunConfig(
                client=client,
                manifest=manifest
            )
        ),
    )
    
    # 步骤 5：处理工件
    print(result.final_output)


asyncio.run(main())
```

### 切换提供商

```python
# 开发：UnixLocal
from agents.sandbox.sandboxes.unix_local import UnixLocalSandboxClient
client = UnixLocalSandboxClient()

# 测试/生产：Docker
from agents.sandbox.sandboxes.docker import DockerSandboxClient
client = DockerSandboxClient(docker_client)

# 代码几乎不需要改动！
result = await Runner.run(
    agent,
    input,
    run_config=RunConfig(
        sandbox=SandboxRunConfig(client=client)
    )
)
```

---

## 八、状态管理

### 三个状态概念

| 状态 | 包含什么 | 恢复什么 | 何时使用 |
|------|---------|---------|---------|
| **RunState** | Harness 状态（对话、工具、审批） | 工作流继续 | 需要代理知道之前做了什么 |
| **Session State** | 沙箱完整状态 | 沙箱环境 | 需要完全恢复执行环境 |
| **Snapshot** | 工作空间文件 | 文件内容 | 新工作空间需要之前的工件 |

### 1. RunState（运行状态）

```python
# 第一次运行
result1 = await Runner.run(agent, "重构代码")

# 获取运行状态
run_state = result1.state

# 保存到数据库
save_to_db(run_state)

# ... 等待审批 ...

# 从数据库加载
loaded_state = load_from_db()

# 继续运行
result2 = await Runner.run(
    agent,
    loaded_state,
    "审批通过，继续添加测试"
)
```

### 2. Session State（会话状态）

```python
# 创建会话
session = await client.create(manifest=manifest)

# 运行
result = await Runner.run(
    agent,
    input,
    run_config=RunConfig(
        sandbox=SandboxRunConfig(session=session)
    )
)

# 序列化会话状态
session_state = client.serialize_session_state(session.state)

# 保存
with open("session-state.json", "w") as f:
    json.dump(session_state, f)

# ... 时间流逝 ...

# 恢复
with open("session-state.json", "r") as f:
    saved_state = json.load(f)

restored_state = client.deserialize_session_state(saved_state)
resumed_session = await client.resume(restored_state)

# 继续工作
result2 = await Runner.run(
    agent,
    input,
    run_config=RunConfig(
        sandbox=SandboxRunConfig(session=resumed_session)
    )
)
```

### 3. Snapshot（快照）

```python
# 创建快照
import tarfile
with tarfile.open("workspace-snapshot.tar.gz", "w:gz") as tar:
    tar.add(session.workspace, arcname=".")

# 从快照创建新会话
manifest = Manifest.from_snapshot("workspace-snapshot.tar.gz")

result = await Runner.run(
    agent,
    input,
    run_config=RunConfig(
        sandbox=SandboxRunConfig(
            client=client,
            manifest=manifest
        )
    )
)
```

### 恢复优先级

```
Runner 解析沙箱会话的顺序：

1. 如果传入 live session → 直接使用
2. 否则，如果从 RunState 恢复 → 从存储的会话状态恢复
3. 否则，如果传入序列化的会话状态 → 从该状态恢复
4. 否则，创建新的沙箱会话
   ├─ 使用 per-run manifest（如果提供）
   └─ 或使用 agent 的 default manifest
```

---

## 九、沙箱记忆

### 会话记忆 vs 沙箱记忆

```
会话记忆（Session Memory）：
├─ SDK 管理
├─ 保存消息历史
├─ 对话的完整记录
└─ 恢复时：重放整个对话

沙箱记忆（Sandbox Memory）：
├─ 沙箱管理
├─ 保存提炼的经验
├─ 可重用的指导
└─ 恢复时：读取经验文件
```

### 使用记忆

```python
from agents.sandbox.capabilities import Memory

agent = SandboxAgent(
    name="学习助手",
    instructions="""你是一个学习助手。
    
每次任务开始：
1. 读取 memories/ 中的经验
2. 根据经验调整方法

每次任务结束：
1. 更新经验文件""",
    capabilities=[Shell(), Filesystem(), Memory()],
)
```

### 记忆文件结构

```
workspace/
├── memories/
│   ├── memory_summary.md      # 记忆摘要
│   ├── MEMORY.md              # 完整记忆
│   ├── raw_memories/          # 原始记忆
│   └── rollout_summaries/     # 运行摘要
└── ...
```

### 记忆模式

```python
# 默认读写
memory = Memory()

# 只读
memory = Memory(
    read=MemoryReadConfig(),
    generate=None
)

# 只生成
memory = Memory(
    read=None,
    generate=MemoryGenerateConfig()
)
```

### 渐进式披露

```
1. SDK 注入 memory_summary.md（摘要）
2. 代理搜索 MEMORY.md（如果需要）
3. 代理打开具体摘要（如果需要更多细节）
```

---

## 十、工作流模式

### 模式 1：多步骤工作流

```python
# 步骤 1：数据预处理
result1 = await Runner.run(
    preprocess_agent,
    "清洗数据",
    run_config=RunConfig(
        sandbox=SandboxRunConfig(client=client, manifest=manifest)
    )
)

# 步骤 2：数据分析（复用会话）
session = result1.sandbox_session
result2 = await Runner.run(
    analyze_agent,
    "分析数据",
    run_config=RunConfig(
        sandbox=SandboxRunConfig(session=session)
    )
)

# 步骤 3：可视化（继续复用）
result3 = await Runner.run(
    visualize_agent,
    "生成图表",
    run_config=RunConfig(
        sandbox=SandboxRunConfig(session=session)
    )
)
```

### 模式 2：人工审查（暂停/恢复）

```python
# 步骤 1：代理完成初步工作
result1 = await Runner.run(agent, "重构代码")

# 步骤 2：暂停，等待审查
session = result1.sandbox_session
frozen_state = client.serialize_session_state(session.state)

# 人工审查
approved = input("审查代码，输入 'y' 继续: ")

if approved == 'y':
    # 步骤 3：恢复，继续工作
    resumed_session = await client.resume(
        client.deserialize_session_state(frozen_state)
    )
    
    result2 = await Runner.run(
        agent,
        "添加测试",
        run_config=RunConfig(
            sandbox=SandboxRunConfig(session=resumed_session)
        )
    )
```

### 模式 3：多代理协作（Handoffs）

```python
from agents import Agent

# 规划员（不需要沙箱）
planner = Agent(
    name="规划员",
    instructions="分析需求，制定计划"
)

# 开发者（需要沙箱）
developer = SandboxAgent(
    name="开发者",
    instructions="实现功能",
    capabilities=[Shell(), Filesystem()]
)

# 设置交接
planner.handoffs = [developer]

# 运行
result = await Runner.run(
    planner,
    "开发一个待办事项应用"
)
```

### 模式 4：服务预览

```python
from agents.sandbox.sandboxes.unix_local import UnixLocalSandboxClientOptions

# 配置暴露的端口
options = UnixLocalSandboxClientOptions(
    exposed_ports=(3000,)
)

agent = SandboxAgent(
    name="Web 助手",
    capabilities=[Shell(), Filesystem()]
)

result = await Runner.run(
    agent,
    "启动 Web 服务器",
    run_config=RunConfig(
        sandbox=SandboxRunConfig(
            client=client,
            options=options
        )
    )
)

# 获取预览 URL
session = result.sandbox_session
endpoint = session.exposed_port_endpoint(port=3000)
print(f"预览 URL: {endpoint.url}")
```

---

## 十一、最佳实践

### 1. 安全第一

```python
# ❌ 错误：把密钥放在 manifest 里
manifest = Manifest(
    environment={
        "API_KEY": "sk-xxx"  # 不要这样做
    }
)

# ✅ 正确：密钥作为运行时配置
result = await Runner.run(
    agent,
    input,
    run_config=RunConfig(
        sandbox=SandboxRunConfig(
            environment={
                "API_KEY": os.environ["API_KEY"]
            }
        )
    )
)
```

### 2. 渐进式采用

```
Level 1：简单任务
→ 直接用 Responses API，不用沙箱

Level 2：偶尔需要 shell
→ 用托管 shell 工具

Level 3：复杂工作流
→ 用沙箱代理

Level 4：生产环境
→ 用 Docker 或托管沙箱（E2B）
```

### 3. 监控回合数

```python
result = await Runner.run(agent, input, max_turns=20)

# 回合数直接影响：
# - API 成本
# - 响应时间
# - Token 消耗
```

### 4. 选择合适的沙箱提供商

```
本地开发：UnixLocalSandboxClient
├─ 快速迭代
├─ 容易调试
└─ 无需容器

测试环境：DockerSandboxClient
├─ 隔离性好
├─ 可复现
└─ 自定义镜像

生产环境：E2B / Modal / 其他托管
├─ 完全托管
├─ 自动扩展
├─ 安全隔离
└─ 快照和恢复
```

### 5. 凭证处理规则

- 凭证不要出现在提示、指令、任务文件、manifest 中
- 使用运行时配置注入
- 挂载存储的凭证要限制作用域
- 标记敏感条目为 ephemeral（临时的）

---

## 十二、完整示例

### 示例：数据分析助手

```python
import asyncio
import os
from agents import Runner
from agents.run import RunConfig
from agents.sandbox import SandboxAgent, SandboxRunConfig, Manifest
from agents.sandbox.capabilities import Shell, Filesystem, Memory, Compaction
from agents.sandbox.entries import File, Dir, GitRepo
from agents.sandbox.sandboxes.unix_local import UnixLocalSandboxClient


async def main():
    # 1. 创建 Manifest
    manifest = Manifest(
        entries={
            "README.md": File(content=b"# 数据分析项目"),
            "src": Dir(),
            "tests": Dir(),
            "templates": GitRepo(
                repo="https://github.com/org/templates",
                ref="main"
            ),
        },
        environment={
            "PYTHONPATH": "/workspace",
        }
    )
    
    # 2. 创建代理
    agent = SandboxAgent(
        name="数据分析助手",
        instructions="""你是一个数据分析专家。
        
每次任务开始：
1. 读取 memories/ 中的经验
2. 检查项目状态
3. 根据经验调整方法

完成任务后：
1. 更新经验文件
2. 生成报告到 output/""",
        capabilities=[
            Shell(),
            Filesystem(),
            Memory(),
            Compaction(),
        ],
        default_manifest=manifest,
    )
    
    # 3. 选择沙箱客户端
    client = UnixLocalSandboxClient()
    
    # 4. 运行时配置（注入凭证）
    run_config = RunConfig(
        sandbox=SandboxRunConfig(
            client=client,
            environment={
                "DATABASE_URL": os.environ.get("DATABASE_URL", ""),
                "API_KEY": os.environ.get("API_KEY", ""),
            }
        )
    )
    
    # 5. 第一次运行
    print("=== 第一次运行 ===")
    result1 = await Runner.run(
        agent,
        "分析 sales.csv 并生成报告",
        run_config=run_config
    )
    
    print(result1.final_output)
    
    # 6. 保存会话状态
    session = result1.sandbox_session
    frozen_state = client.serialize_session_state(session.state)
    
    # 7. 暂停等待审查
    print("\n=== 等待审查 ===")
    input("审查完成后按回车继续...")
    
    # 8. 恢复会话
    print("\n=== 恢复会话 ===")
    resumed_session = await client.resume(
        client.deserialize_session_state(frozen_state)
    )
    
    # 9. 继续工作
    result2 = await Runner.run(
        agent,
        "审查通过，添加单元测试",
        run_config=RunConfig(
            sandbox=SandboxRunConfig(session=resumed_session)
        )
    )
    
    print(result2.final_output)
    
    # 10. 清理
    await client.delete(resumed_session)


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 总结

### 核心要点

1. **沙箱是什么**：隔离的执行环境，让代理能真正"动手做事"
2. **架构设计**：Harness（控制）vs Compute（执行）分离
3. **何时使用**：需要操作文件、运行命令、生成工件、暴露端口时
4. **核心组件**：SandboxAgent、Manifest、Capabilities、Client、Session
5. **状态管理**：RunState（工作流）、Session State（环境）、Snapshot（文件）
6. **记忆系统**：跨运行学习，保存提炼的经验
7. **工作流模式**：多步骤、暂停恢复、多代理协作、服务预览

### 选择指南

```
需要什么？
│
├─ 执行命令？ → Shell()
├─ 操作文件？ → Filesystem()
├─ 初始化工作空间？ → Manifest
├─ 跨运行记忆？ → Memory()
├─ 专业技能？ → Skills()
├─ 暂停恢复？ → Session Resume
├─ 不同环境？ → 选择合适的 Sandbox Client
└─ 多代理协作？ → Handoffs 或 Nested Tools
```

### 一句话总结

沙箱代理让 AI 从"顾问"变成"执行者"，是构建生产级代理系统的关键能力。通过 Harness 和 Compute 的分离，实现了安全、灵活、可扩展的代理执行环境。

---

**文档版本**：1.0  
**最后更新**：2026-08-15  
**参考**：OpenAI Agents SDK 官方文档
