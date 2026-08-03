# DaemonSet 与节点级守护进程

> 本文从理论到实战部署 Node Exporter，并串联 StatefulSet 笔记中 Flannel 排查的真实经历。
> 最后更新：2026-08-03

---

## 目录

**第一部分：理论基础**
1. [DaemonSet 是什么](#1-daemonset-是什么)
2. [三大特性](#2-三大特性)
3. [与 Deployment / StatefulSet 对比](#3-与-deployment--statefulset-对比)
4. [什么场景需要 DaemonSet](#4-什么场景需要-daemonset)

**第二部分：怎么使用**
5. [基本结构](#5-基本结构)
6. [调度控制：nodeSelector / nodeAffinity / tolerations](#6-调度控制nodeselector--nodeaffinity--tolerations)
7. [更新策略](#7-更新策略)
8. [常用操作](#8-常用操作)

**第三部分：实战部署 Node Exporter**
9. [架构设计](#9-架构设计)
10. [完整配置文件（逐行注释）](#10-完整配置文件逐行注释)
11. [部署与验证](#11-部署与验证)

**第四部分：你早见过的 DaemonSet（Flannel 真实案例）**
12. [回顾 Flannel 排查](#12-回顾-flannel-排查)
13. [DaemonSet 在其中的作用](#13-daemonset-在其中的作用)

**第五部分：关键知识点**
14. [hostNetwork / hostPID 的意义](#14-hostnetwork--hostpid-的意义)
15. [DaemonSet 调度机制：taint 与 toleration](#15-daemonset-调度机制taint-与-toleration)
16. [三大工作负载总图](#16-三大工作负载总图)
17. [概念关系总图](#17-概念关系总图)

---

# 第一部分：理论基础

## 1. DaemonSet 是什么

**DaemonSet（守护进程集）= 确保每个（符合条件的）节点上都运行一个 Pod 副本的控制器。**

延续 StatefulSet 笔记里的"公司组织架构"心智模型：

| 控制器 | 心智模型 | 数量由谁决定 |
|---|---|---|
| **Deployment** | 某部门"我要 5 个客服"，不关心坐在哪个工位 | 你写 `replicas: 5` |
| **StatefulSet** | "我要 6 个有工号的员工"，有序入职、各有专属抽屉 | 你写 `replicas: 6` |
| **DaemonSet** | "每个分店**各派 1 个**驻店保安"，新开分店自动派，关店自动撤 | **节点数**（不用写 replicas） |

关键区别：**DaemonSet 的副本数 = 节点数，你说了不算，集群节点数说了算。**

```
Deployment 思维:  "给我 N 个 Pod,随便放哪"
DaemonSet 思维:   "每个节点上必须有一个,一个不能多,一个不能少"
```

> 💡 你其实早就见过 DaemonSet 了。StatefulSet 笔记里排查 Flannel 网络问题时，
> 执行 `kubectl delete pod -n kube-flannel kube-flannel-ds-jbs8z` 删掉 k8s3 的 Flannel Pod，
> 然后"等 DaemonSet 重建"——那个自动补上 Pod 的就是 DaemonSet。本文最后会详细回顾。

---

## 2. 三大特性

### 2.1 一节点一 Pod（默认）
每个节点跑且仅跑一个副本。这是它和 Deployment 最本质的区别——Deployment 的 N 个副本可能全挤在一个节点上，DaemonSet 是铁打的"一节点一个"。

### 2.2 跟随节点动态伸缩
- **新节点加入集群** → 自动在上面起一个 Pod
- **节点被移除** → 上面的 Pod 自动回收

```
集群有 k8s1/k8s2/k8s3 三个节点
  ↓
DaemonSet 自动起 3 个 Pod,每个节点一个
  ↓
加入 k8s4 节点  →  自动在 k8s4 起 1 个 Pod(共 4 个)
  ↓
移除 k8s2 节点  →  k8s2 上的 Pod 自动消失(共 3 个)
```

这正是 Flannel 排查时的体验：删掉 k8s3 的 Flannel Pod 后，不用手动重建，DaemonSet 控制器发现"k8s3 上少了一个"，立刻补上。

### 2.3 不需要 `replicas` 字段
写 `replicas` 没意义（DaemonSet 会忽略它），数量由节点数决定。

---

## 3. 与 Deployment / StatefulSet 对比

| 特性 | Deployment | StatefulSet | DaemonSet |
|---|---|---|---|
| 副本数 | 你指定 `replicas` | 你指定 `replicas` | **= 节点数**（无需指定） |
| Pod 放哪 | 调度器随便放 | 调度器随便放（但带身份） | **每节点各一个** |
| Pod 名 | 随机（`app-5f8b-x2k9p`） | 固定序号（`mysql-0`） | 随机（`flannel-xxxxx`） |
| 启停顺序 | 无序 | 有序（0→1→2） | 无序，但每节点独立 |
| 独立存储 | ❌ 共享 PVC | ✅ 每 Pod 独立 PVC | 通常用 `hostPath`（挂宿主机） |
| 网络身份 | 共享 Service VIP | 每 Pod 独立 DNS | 常用 `hostNetwork`（宿主机网络） |
| 典型场景 | Web 服务、API | 数据库、消息队列 | 网络插件、日志、监控 |

**一句话记忆**：
- Deployment 关心**副本数**（要几个）
- StatefulSet 关心**身份**（谁是谁，数据在哪）
- DaemonSet 关心**节点**（每台机器都得有）

---

## 4. 什么场景需要 DaemonSet

**判断标准：凡是"每个节点都必须有一个、且要和这个节点本身打交道"的需求，就用 DaemonSet。**

| 场景 | 代表组件 | 为什么必须每节点一个 |
|---|---|---|
| **网络插件** | Flannel、Calico、Cilium | 要给本节点配 flannel.1 接口、路由表，跨节点流量靠本节点转发 |
| **存储插件** | CSI node plugin、local-provisioner | 要挂载本节点的磁盘给 Pod 用 |
| **日志采集** | Fluentd、Filebeat、Promtail | 要读本节点 `/var/log` 下的容器日志 |
| **监控采集** | Node Exporter | 要采集本节点的 CPU/内存/磁盘指标 |
| **安全/运行时** | Falco、Tetragon | 要监控本节点的系统调用、文件访问 |

共同点：**全部是基础设施类组件**，而不是业务应用。

**反向判断**：
- 你的应用需要"3 个副本做负载均衡" → **Deployment**
- 你的应用需要"每台机器都得有，少一台都不行" → **DaemonSet**

---

# 第二部分：怎么使用

## 5. 基本结构

骨架和 Deployment 几乎一样，区别就两点：**`kind: DaemonSet` + 没有 `replicas`**。

```yaml
apiVersion: apps/v1
kind: DaemonSet                  # 👈 就这里不一样
metadata:
  name: my-daemon
spec:
  selector:                      # 和 Deployment 一样,必须和 template.labels 一致
    matchLabels:
      app: my-daemon
  template:
    metadata:
      labels:
        app: my-daemon           # 👈 要和 selector 一致
    spec:
      containers:
      - name: agent
        image: busybox
        command: ["sleep", "3600"]
```

> 和 StatefulSet 一样，`selector.matchLabels` 必须和 `template.labels` 严格一致，
> 否则 DaemonSet 无法认领它创建的 Pod。详见 StatefulSet 笔记的[关联关系总图](./StatefulSet.md#18-概念关系总图)。

---

## 6. 调度控制：nodeSelector / nodeAffinity / tolerations

默认情况下，DaemonSet 想跑在所有节点上。但你可以用三种方式收窄或放宽范围：

| 方式 | 作用 | 典型用法 |
|---|---|---|
| `nodeSelector` | 只跑在带某 label 的节点（最简单） | 只跑在 SSD 节点 |
| `nodeAffinity` | 更灵活的节点选择（支持多种条件、软约束） | 只跑在 x86 且非 ARM 节点 |
| `tolerations` | 允许跑在被"污染"（taint）的节点 | 允许跑在 master 节点 |

```yaml
spec:
  template:
    spec:
      # ① nodeSelector：只跑在带 disktype=ssd 的节点
      nodeSelector:
        disktype: ssd

      # ② tolerations：容忍所有 taint（连 master 也跑）
      tolerations:
      - operator: Exists           # 容忍一切污染
```

> ⚠️ **重点**：master（control-plane）节点默认带有
> `node-role.kubernetes.io/control-plane:NoSchedule` 这个 taint（污染），
> 普通 Pod 不让调度上去。所以默认 DaemonSet 也不会去 master 上跑——
> 除非你显式写 toleration 容忍它。
>
> 这就是为什么你之前的环境里 Flannel 只在 k8s2、k8s3 上，
> 而 CoreDNS（Deployment）在 k8s1 上——CoreDNS 不需要每节点一个，调度器把它放 master 也没问题；
> Flannel（DaemonSet）默认避开 master 的 taint。

---

## 7. 更新策略

```yaml
spec:
  updateStrategy:
    type: RollingUpdate           # 默认值。更新镜像时逐个节点替换 Pod
    rollingUpdate:
      maxUnavailable: 1           # 最多同时有 1 个节点的 Pod 处于不可用（默认 1）
  # type: OnDelete                # 另一种：你手动删 Pod 才更新,适合需要精细控制的场景
```

| 策略 | 行为 | 适用 |
|---|---|---|
| **RollingUpdate**（默认） | 改镜像后，逐个节点替换 Pod | 大多数情况 |
| **OnDelete** | 改镜像后不自动更新，等你手动删 Pod 才重建 | 需要逐台手动验证的基础设施 |

`maxUnavailable` 控制并发度：值越大，更新越快，但同一时刻"没有守护进程"的节点越多。对网络插件这类核心组件，通常保持 `1`，慢慢来。

---

## 8. 常用操作

```bash
# 查看所有 DaemonSet（注意 -A，系统组件多在 kube-system / kube-flannel）
kubectl get daemonset -A
kubectl get ds -A                  # ds 是 daemonset 的简写

# 看某个 DaemonSet 的 Pod 分布（验证每节点一个）
kubectl get pod -n kube-flannel -o wide

# 更新镜像，触发滚动更新
kubectl set image daemonset/node-exporter node-exporter=prom/node-exporter:v1.8.0 -n monitoring

# 看滚动更新进度
kubectl rollout status daemonset/node-exporter -n monitoring

# 删除 DaemonSet（默认级联删除它创建的 Pod）
kubectl delete daemonset node-exporter -n monitoring
# 保留 Pod（孤儿模式）
kubectl delete daemonset node-exporter -n monitoring --cascade=orphan
```

---

# 第三部分：实战部署 Node Exporter

## 9. 架构设计

Node Exporter 是 Prometheus 生态里最经典的 DaemonSet 用例——职责就是"采集每台机器的主机指标（CPU/内存/磁盘/网络）"，天然适合每节点一个。

```
┌─────────────────────────────────────────────┐
│         集群(3 个节点)                       │
│                                             │
│   k8s1(master)  k8s2(worker)  k8s3(worker)  │
│       │             │             │         │
│   node-exporter node-exporter node-exporter │  ← DaemonSet 自动起 3 个
│       │             │             │         │
│       :9100         :9100         :9100     │  ← 每个用宿主机网络暴露端口
│       │             │             │         │
│       └─────────────┴─────────────┘         │
│                     │                       │
│              Prometheus 统一抓取             │
└─────────────────────────────────────────────┘
```

---

## 10. 完整配置文件（逐行注释）

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter            # 【关联1】必须和 template.labels 一致
  template:
    metadata:
      labels:
        app: node-exporter          # 【关联1对端】
    spec:
      hostNetwork: true             # 👈 直接用宿主机网络,绕过 CNI,采集更准
      hostPID: true                 # 👈 能看到宿主机进程命名空间,采进程指标
      tolerations:                  # 👈 放行 master 节点,保证每台都采
      - operator: Exists            #    容忍所有 taint(连 master 也采)
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.7.0
        ports:
        - containerPort: 9100
          hostPort: 9100            # 👈 把端口映射到宿主机,Prometheus 好抓
        resources:                  # 守护进程要克制,别抢业务资源
          limits:
            cpu: 200m
            memory: 100Mi
          requests:
            cpu: 100m
            memory: 50Mi
        volumeMounts:
        - name: proc                # 【关联2左半】挂宿主机 /proc,采 CPU/内存
          mountPath: /host/proc
          readOnly: true
        - name: sys                 # 【关联2左半】挂宿主机 /sys,采磁盘
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc                  # 【关联2右半】声明卷来自宿主机路径
        hostPath:
          path: /proc
          type: Directory
      - name: sys
        hostPath:
          path: /sys
          type: Directory
```

### 这个案例里的几个"DaemonSet 特有套路"

| 配置 | 作用 | 为什么 DaemonSet 常用 |
|---|---|---|
| `hostNetwork: true` | 不走 Pod 网络，直接用宿主机 IP | 网络插件自己（Flannel）就是这么干的：网络插件还没起来时 Pod 网络根本不通，只能用宿主机网络 |
| `hostPath` 挂载 | 把宿主机 `/proc`、`/sys` 挂进容器 | 要采宿主机指标，必须能读到宿主机的内核数据，这是"和节点本身打交道"的体现 |
| `tolerations: Exists` | 容忍所有 taint | 基础设施监控不能漏机器，连 master 也要采 |
| `resources` 限资源 | 给守护进程设上限 | 守护进程和业务 Pod 抢资源，不限制可能把节点搞挂 |

---

## 11. 部署与验证

```bash
# 1. 创建命名空间
kubectl create namespace monitoring

# 2. 部署
kubectl apply -f node-exporter.yaml

# 3. 验证：每个节点各一个 Pod
kubectl get pod -n monitoring -o wide
# 期望：NODE 列显示分布在所有节点上,每个节点一个 Running 的 Pod

# 4. 验证：能拿到指标（任选一个节点 IP）
curl <节点IP>:9100/metrics | head
```

预期输出类似：

```
NAME                   READY   STATUS    NODE       AGE
node-exporter-abc12    1/1     Running   k8s1       1m
node-exporter-def34    1/1     Running   k8s2       1m
node-exporter-ghi56    1/1     Running   k8s3       1m
```

---

# 第四部分：你早见过的 DaemonSet（Flannel 真实案例）

## 12. 回顾 Flannel 排查

在 StatefulSet 笔记的[坑 6：DNS 解析失败](./StatefulSet.md#坑-6dns-解析失败flannel-跨节点网络断裂)中，你遇到过这样的现象：

- k8s3 上跨节点网络不通，DNS 解析失败
- `kubectl get pods -n kube-flannel -o wide` 显示 Flannel **在 3 个节点都 Running**
- 但 k8s3 的 `flannel.1` 接口**没有 IPv4 地址**，缺少跨节点路由

当时的修复是：

```bash
kubectl delete pod -n kube-flannel kube-flannel-ds-jbs8z
```

注意这个 Pod 名字：**`kube-flannel-ds-jbs8z`**——那个 `ds` 就是 DaemonSet 的缩写。它是 Flannel 的 DaemonSet `kube-flannel-ds` 创建出来的 Pod。

---

## 13. DaemonSet 在其中的作用

### 13.1 为什么 Flannel 用 DaemonSet

Flannel 是网络插件，必须给**每个节点**配置 `flannel.1` 接口和跨节点路由表。少配一个节点，那个节点上的 Pod 就连不出去——这正是你踩的坑。

所以 Flannel 铁定用 DaemonSet：保证每节点一个，新节点加入自动配。

### 13.2 为什么 `kubectl delete pod` 能"重启" Flannel

```
你执行: kubectl delete pod kube-flannel-ds-jbs8z (k8s3 上的)
          ↓
   Pod 被删除
          ↓
   DaemonSet 控制器检测到:"k8s3 上少了一个 Pod!不符合期望状态"
          ↓
   立即在 k8s3 重新创建一个 Flannel Pod
          ↓
   新 Pod 重新初始化 flannel.1 接口、重写路由表
          ↓
   网络恢复 ✅
```

这正是 DaemonSet **调谐循环（Reconciliation Loop）**的体现（和 [Operator 的调谐循环](./StatefulSet.md#63-核心机制reconciliation-loop调谐循环)是同一个思想）：

> 持续对比"期望状态（每节点一个 Pod）" vs "实际状态（k8s3 没有 Pod）"，不一致就修复。

### 13.3 重要教训：DaemonSet 的 Pod 可以随便删

```
Deployment  删 Pod  →  控制器补一个新的(副本数不变)
StatefulSet 删 Pod  →  控制器用同名 Pod 补回(身份不变,数据还在)
DaemonSet   删 Pod  →  控制器在该节点补一个新的(每节点一个不变)
```

**三种控制器都会自动重建被删的 Pod**。所以当你怀疑某个基础设施 Pod 状态异常（Running 但实际没初始化好，就像 k8s3 的 Flannel），直接删掉让它重建，是最常用、最安全的排查手段。

> 这也是为什么排查 Flannel 时不需要重启整个节点——只需删 Pod，DaemonSet 会重新拉起一个干净的实例。

---

# 第五部分：关键知识点

## 14. hostNetwork / hostPID 的意义

DaemonSet 经常碰到这三个 `host*` 配置：

| 配置 | 含义 | 典型用途 |
|---|---|---|
| `hostNetwork: true` | Pod 直接用宿主机网络栈，不分配 Pod IP | Flannel、Node Exporter（网络插件还没起时 Pod 网络不通） |
| `hostPID: true` | Pod 看得到宿主机的进程命名空间 | 监控组件要看宿主机进程 |
| `hostIPC: true` | 共享宿主机 IPC（进程间通信） | 较少用，某些共享内存场景 |

**为什么业务应用不用 `hostNetwork`？**
- 端口会和宿主机其他进程冲突
- 失去 Service 负载均衡的能力
- 一台机器上想跑多个副本就冲突了

但 DaemonSet 不在乎这些——它本来就"每节点一个"，不存在端口冲突，且恰恰需要直接接触宿主机。

---

## 15. DaemonSet 调度机制：taint 与 toleration

### 15.1 taint（污染）和 toleration（容忍）的关系

```
节点打 taint:  "我这是 master,普通 Pod 别来" (NoSchedule)
                  ↓
普通 Pod:       没有 toleration → 不被调度到 master
                  ↓
DaemonSet Pod:  写了 toleration → 可以调度到 master
```

### 15.2 常见 taint

| taint | 含义 |
|---|---|
| `node-role.kubernetes.io/control-plane:NoSchedule` | master 节点，默认不让普通 Pod 调度 |
| `node.kubernetes.io/not-ready:NoSchedule` | 节点未就绪，暂时别调度 |
| `node.kubernetes.io/disk-pressure` | 磁盘压力 |

### 15.3 让 DaemonSet 跑在所有节点（含 master）

```yaml
tolerations:
- operator: Exists               # 容忍一切 taint
```

或者更精细：

```yaml
tolerations:
- key: node-role.kubernetes.io/control-plane
  operator: Exists               # 只容忍 master 的 taint
```

Node Exporter 案例用了 `Exists`（容忍一切），因为监控要覆盖所有机器。

---

## 16. 三大工作负载总图

```
                    ┌─ 业务应用(关心副本数、负载均衡)────► Deployment
                    │
K8s 工作负载 ───────┼─ 数据应用(关心身份、存储、有序)────► StatefulSet
                    │
                    └─ 基础设施(关心节点本身)──────────► DaemonSet
```

| 维度 | Deployment | StatefulSet | DaemonSet |
|---|---|---|---|
| 核心诉求 | 弹性伸缩 + 负载均衡 | 稳定身份 + 独立存储 | 每节点覆盖 |
| 副本数 | 你定 | 你定 | 节点数 |
| 数据 | 无状态 | 每 Pod 独立 PVC | 多用 hostPath |
| 网络 | Service VIP | Headless + 独立 DNS | 常用 hostNetwork |
| 删 Pod 后 | 重建随机名 Pod | 重建同名 Pod（数据还在） | 在该节点重建 Pod |
| 例子 | Nginx、API 服务 | MySQL、Redis Cluster | Flannel、Node Exporter |

---

## 17. 概念关系总图

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes 工作负载层级                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Pod (最小单元)                                          │
│    ↑                                                    │
│    ├── Deployment (无状态副本,关心"要几个")              │
│    │                                                     │
│    ├── StatefulSet (有状态,关心"身份+存储")              │
│    │     └── 必须绑定 Headless Service                   │
│    │                                                     │
│    └── DaemonSet (节点级守护,关心"每台都有")             │
│          ├── 数量 = 节点数(无需 replicas)                │
│          ├── 常用 hostNetwork / hostPath                 │
│          └── 用 tolerations 放行 master                  │
│                                                         │
│  调度控制(三种控制器都可能用):                            │
│    nodeSelector ──► 按 label 选节点                      │
│    nodeAffinity ──► 更灵活地选节点                       │
│    tolerations ───► 容忍 taint(放行被污染的节点)         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 三种控制器的统一规律

和 StatefulSet 笔记[最后的总结](./StatefulSet.md#18-概念关系总图)一致——**K8s 里很多东西都是靠"名字"和"label"互相找的**：

| 关联类型 | 例子 | 怎么对上 |
|---|---|---|
| **内部 label 配对** | `selector.matchLabels` ↔ `template.labels` | label 一致 |
| **节点选择** | `nodeSelector` ↔ 节点的 label | label 一致 |
| **容忍关系** | `tolerations.key` ↔ 节点的 `taint.key` | key 一致 |

> DaemonSet 和 Deployment 在 YAML 结构上几乎相同，区别只在 `kind` 和"没有 replicas"。
> 真正让 DaemonSet 特殊的是它的**调度逻辑**（每节点一个）和**常见配置**（hostNetwork/hostPath/tolerations）。

---

## 参考资料

- [Kubernetes 官方文档 - DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Kubernetes 官方文档 - Taints 和 Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Node Exporter 项目](https://github.com/prometheus/node_exporter)
- [Flannel 架构](https://github.com/flannel-io/flannel)
- 本文串联的前序笔记：[StatefulSet 与有状态工作负载](./StatefulSet.md)

---

*最后更新：2026-08-03*
