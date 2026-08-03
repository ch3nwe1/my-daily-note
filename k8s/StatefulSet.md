# StatefulSet 与有状态工作负载

> 本文记录了从理论到实战部署 Redis Cluster 的完整学习过程，包含大量真实踩坑经验。
> 最后更新：2026-08-03

---

## 目录

**第一部分：理论基础**
1. [Service 基础概念](#1-service-基础概念)
2. [Headless Service（无头服务）](#2-headless-service无头服务)
3. [StatefulSet（有状态控制器）](#3-statefulset有状态控制器)
4. [为什么 StatefulSet 必须用 Headless Service](#4-为什么-statefulset-必须用-headless-service)
5. [PV / PVC / StorageClass（存储三件套）](#5-pv--pvc--storageclass存储三件套)
6. [Operator 模式](#6-operator-模式)

**第二部分：实战部署 Redis Cluster**
7. [架构设计](#7-架构设计)
8. [完整配置文件（逐行注释）](#8-完整配置文件逐行注释)
9. [部署步骤](#9-部署步骤)

**第三部分：踩坑实录（真实排查全过程）**
10. [坑 1：PV 配置拼写错误 `cpacity`](#坑-1pv-配置拼写错误-cpacity)
11. [坑 2：YAML 缩进错误导致解析失败](#坑-2yaml-缩进错误导致解析失败)
12. [坑 3：PVC 绑不上 PV（`unbound immediate`）](#坑-3pvc-绑不上-pvunbound-immediate)
13. [坑 4：`redis.conf` 找不到（ConfigMap 未绑定）](#坑-4redisconf-找不到configmap-未绑定)
14. [坑 5：PV 卡在 Released 状态](#坑-5pv-卡在-released-状态)
15. [坑 6：DNS 解析失败（Flannel 跨节点网络断裂）](#坑-6dns-解析失败flannel-跨节点网络断裂)

**第四部分：关键知识点**
16. [为什么删除 StatefulSet 不删除 PVC](#16-为什么删除-statefulset-不删除-pvc)
17. [StorageClass 的本质：标签 + 行为说明书](#17-storageclass-的本质标签--行为说明书)
18. [概念关系总图](#18-概念关系总图)

---

# 第一部分：理论基础

## 1. Service 基础概念

### 1.1 Service 是什么

**Service = 稳定的"入口" + 自动的"分发规则"**

解决的问题：
- Pod IP 随机变化，无法稳定访问
- 多个 Pod 之间如何负载均衡
- Pod 挂了如何感知并剔除
- 扩缩容后如何自动更新后端列表

### 1.2 VIP（Virtual IP，虚拟 IP）

VIP 是一个**不真实存在**于任何网卡上的 IP，只是 kube-proxy 设置的 iptables 规则中的"钩子"。

类比：公司的 400 客服电话 -> 不对应具体电话机，自动转接到值班员工。

访问 VIP 的流程：
```
请求 -> iptables 规则拦截（"这是 Service 的 VIP"）
     -> 负载均衡算法选一个后端 Pod
     -> 转发到 Pod 真实 IP
```

### 1.3 Service 类型

| 类型 | 说明 | 适用场景 |
|---|---|---|
| **ClusterIP** | 默认，只能在集群内部访问 | 服务间互相调用 |
| **LoadBalancer** | 云厂商分配公网 IP/ELB | 对外暴露 Web 应用 |
| NodePort | 在每个节点上暴露固定端口 | 开发测试 |
| ExternalName | 外部 DNS 的 CNAME | 连接外部服务 |

### 1.4 心智模型：公司组织架构

- **Pod** = 具体的员工（会离职、入职）
- **Deployment** = 某个部门（"我要 5 个客服"--负责招人/裁人）
- **Service** = 对外的客服电话（号码永远不变，谁在岗就谁接）

---

## 2. Headless Service（无头服务）

### 2.1 定义

**Headless Service = `clusterIP: None` 的 Service**

- 没有 VIP
- 不做负载均衡
- DNS 直接返回后端所有 Pod 的 IP 列表（或每个 Pod 独立的 DNS 记录）

### 2.2 与普通 Service 对比

```
普通 Service:
  client -> VIP -> kube-proxy -> Pod A / Pod B / Pod C (负载均衡)

Headless Service:
  client -> DNS 查询 -> [Pod A IP, Pod B IP, Pod C IP] (客户端自己选)
```

### 2.3 配置

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless-svc
spec:
  clusterIP: None           # 👈 关键！
  selector:
    app: myapp
  ports:
  - port: 3306
```

### 2.4 与 StatefulSet 结合：每 Pod 独立 DNS

当 Headless Service 被 StatefulSet 引用时，每个 Pod 拿到一条稳定 DNS：

```
<pod-name>.<svc-name>.<namespace>.svc.cluster.local

mysql-0.mysql-headless.default.svc.cluster.local  -> 10.0.0.5
mysql-1.mysql-headless.default.svc.cluster.local  -> 10.0.0.6
mysql-2.mysql-headless.default.svc.cluster.local  -> 10.0.0.7
```

---

## 3. StatefulSet（有状态控制器）

### 3.1 定义

**StatefulSet = 给 Pod 提供"稳定身份 + 独立存储 + 有序部署"的控制器**

与 Deployment 对比：

| 特性 | Deployment | StatefulSet |
|---|---|---|
| Pod 名 | 随机（`app-5f8b-x2k9p`） | 固定序号（`mysql-0`, `mysql-1`） |
| 启停顺序 | 无序 | 有序（0->1->2 启动，2->1->0 停止） |
| 存储 | 共享 PVC，Pod 重建后丢失 | 每个 Pod 独享 PVC，重建后仍绑定同一份数据 |
| 网络身份 | 共享 Service VIP | 每个 Pod 独立 DNS |
| 适用场景 | Web 服务、API | 数据库、消息队列 |

### 3.2 三个核心承诺

1. **稳定的网络身份**（Stable Network Identity）
2. **稳定的存储**（Stable Storage）
3. **有序部署/扩缩/删除**

---

## 4. 为什么 StatefulSet 必须用 Headless Service

### 4.1 根本原因

普通 ClusterIP Service 的 VIP **抹掉了 Pod 身份**--客户端无法指名道姓地访问某个具体 Pod。

但对有状态应用（MySQL 主从、Kafka、Raft 集群），客户端**必须知道数据在哪台**：
- MySQL：写操作必须打到主库
- Kafka：每个 partition 的 leader 在特定 broker 上
- Redis Cluster：每个 slot 归特定节点管

### 4.2 Headless Service 解决了什么

1. **给每个 Pod 发独立 DNS**：`<pod-name>.<svc-name>.<ns>.svc.cluster.local`
2. **主域名解析返回所有 Pod IP**：客户端自己决定连谁

### 4.3 因果链

```
StatefulSet (serviceName=mysql)
  ↓
控制器为每个 Pod 注入 hostname=mysql-N, subdomain=mysql
  ↓
Pod 创建（带 hostname/subdomain）
  ↓
kube-dns 检查 subdomain 对应的 Service
  ├── Service 有 clusterIP -> DNS 只返回 VIP ❌ (Pod 身份丢失)
  └── Service 是 headless  -> DNS 返回 <pod>.<svc>.<ns>.svc... ✅
```

### 4.4 如果不用 Headless 会怎样

- ❌ Pod 专属 DNS 不存在
- ❌ Pod 间无法用稳定名字互相访问
- ❌ 有状态协议（主从选举、分片路由、Raft 投票）全部失效
- ❌ StatefulSet 失去"稳定网络身份"承诺

---

## 5. PV / PVC / StorageClass（存储三件套）

### 5.1 三者关系

```
PVC（存储声明）         PV（实际存储）
  "我要 1Gi 存储"    匹配    "我有 1Gi 空间"
       ↑                          ↑
       └── storageClassName ──────┘
              靠这个"标签"匹配
                     ↑
            StorageClass（存储类）
              定义这类存储的行为
```

### 5.2 匹配条件（不看名字！）

| 匹配条件 | 说明 |
|---|---|
| `accessModes` | RWO / ROX / RWX 必须兼容 |
| `storage` | PV 容量 >= PVC 请求 |
| `storageClassName` | 必须一致 |
| `nodeAffinity`（local 卷） | PV 所在节点必须和 Pod 调度节点一致 |

### 5.3 静态 vs 动态 provisioning

**静态（手动建 PV）**：
```
管理员手动建 PV -> PVC 匹配现成 PV -> 绑定
StorageClass 此时像个"带绑定规则的标签"
```

**动态（自动建 PV）**：
```
只写 PVC -> StorageClass 的 provisioner 自动创建 PV -> 绑定
StorageClass 此时像个"造盘子的配方"
```

### 5.4 volumeBindingMode

| 模式 | 行为 | 适用 |
|---|---|---|
| **Immediate** | PVC 创建后立即尝试绑定 PV | 云盘等不关心节点位置的存储 |
| **WaitForFirstConsumer** | 等 Pod 调度到某节点后，再绑定该节点上的 PV | local 卷（必须和 Pod 同节点） |

### 5.5 reclaimPolicy（回收策略）

| 策略 | 删 PVC 后 PV 的行为 | 数据 |
|---|---|---|
| **Retain** | PV 变 Released，**数据保留** | ✅ 还在 |
| **Delete** | PV 自动删除，**磁盘数据删除** | ❌ 没了 |

---

## 6. Operator 模式

### 6.1 定义

**Operator = 一段自定义的控制逻辑 + 一组自定义资源（CRD）**

把运维知识编码成软件，自动运维复杂应用。

### 6.2 为什么需要 Operator

StatefulSet 只解决"起 Pod + 绑存储"，真正的运维还需要：

| 场景 | StatefulSet 能做 | Operator 做 |
|---|---|---|
| 主库挂了 | 重启 Pod | 自动提升从库为主库、改路由 |
| 扩容 | 起新 Pod | 自动配置复制、同步数据 |
| 备份 | ❌ | 定时 dump 到 S3 |
| 滚动升级 | 重启 Pod | 按顺序、切流量、检查兼容性 |

### 6.3 核心机制：Reconciliation Loop（调谐循环）

```
读取用户写的"期望状态" (CR YAML)
  ↓
读取集群里的"实际状态" (Pod/Service/PVC/...)
  ↓
对比：期望 vs 实际
  ├── 一致 -> 等待
  └── 不一致 -> 执行动作让实际=期望 -> 再次对比（持续循环）
```

---

# 第二部分：实战部署 Redis Cluster

## 7. 架构设计

```
┌─────────────────────────────────────────┐
│        Redis Cluster (6 pods)           │
│   3 Masters    +    3 Replicas          │
│                                         │
│   redis-0 (M1) ─── redis-3 (R1)         │
│   redis-1 (M2) ─── redis-4 (R2)         │
│   redis-2 (M3) ─── redis-5 (R3)         │
└─────────────────────────────────────────┘
```

最少需要 **3 主 + 3 从 = 6 个 Pod**（Redis Cluster 最低要求）。

### 部署环境

- 3 台虚拟机：k8s1（master）、k8s2（worker）、k8s3（worker）
- PV 只建在 worker 节点（k8s2、k8s3），master 不跑有状态应用
- 每个 worker 节点放 3 个 PV

---

## 8. 完整配置文件（逐行注释）

### 8.1 StorageClass：redis-sc.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner   # 不自动创建 PV，用现成的
volumeBindingMode: WaitForFirstConsumer     # 等 Pod 调度后再绑定（local 卷必须）
```

### 8.2 ConfigMap：redis-config.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster-config
data:
  redis.conf: |
    cluster-enabled yes                # 开启集群模式
    cluster-config-file nodes.conf     # 集群拓扑信息文件（相对路径，写到 dir 目录）
    cluster-node-timeout 5000          # 节点超时（毫秒）
    appendonly yes                     # 开启 AOF 持久化
    dir /data                          # 工作目录（显式声明，文件都写这里）
    cluster-announce-port 6379         # 集群通信端口
    cluster-announce-bus-port 16379    # 集群 bus 端口（= 客户端端口 + 10000）
    bind 0.0.0.0
    protected-mode no
```

> **关键**：`cluster-config-file nodes.conf` 是相对路径，相对于 `dir /data`。
> 所以 `nodes.conf` 和 AOF 文件都写到 `/data`，而 `/data` 挂了持久卷。
> 如果不挂持久卷，Pod 重启后集群拓扑和数据全丢。

### 8.3 Headless Service：redis-headless.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster-headless
spec:
  clusterIP: None                    # 👈 无头 Service，给每个 Pod 发独立 DNS
  selector:
    app: redis-cluster               # 靠 label 找到 Redis Pod
  ports:
  - name: client
    port: 6379
    targetPort: 6379
  - name: gossip                     # 集群节点间通信的 bus 端口
    port: 16379
    targetPort: 16379
```

### 8.4 Client Service：redis-client.yaml（可选）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
spec:
  type: ClusterIP
  selector:
    app: redis-cluster
  ports:
  - port: 6379
    targetPort: 6379
```

### 8.5 PV：redis-pv.yaml（6 个，分布在 k8s2 和 k8s3）

```yaml
# k8s2 上的 3 个 PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-pv-0
spec:
  capacity:
    storage: 1Gi
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /tmp/redis-data-0
  nodeAffinity:                        # local 卷必须声明在哪个节点
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s2
---
# ... redis-pv-1、redis-pv-2 同样在 k8s2 ...
---
# k8s3 上的 3 个 PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-pv-3
spec:
  capacity:
    storage: 1Gi
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /tmp/redis-data-3
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s3
---
# ... redis-pv-4、redis-pv-5 同样在 k8s3 ...
```

> ⚠️ **YAML 缩进极其重要**，详见[坑 2](#坑-2yaml-缩进错误导致解析失败)。

### 8.6 StatefulSet：redis-statefulset.yaml（完整注释版）

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  # 【关联1】绑定 Headless Service，名字必须一致
  # 让每个 Pod 拿到独立 DNS：redis-cluster-N.redis-cluster-headless.default.svc.cluster.local
  serviceName: redis-cluster-headless

  replicas: 6                             # 3 主 + 3 从

  # 【关联2】selector 必须和 template.labels 一致，StatefulSet 才能认领 Pod
  selector:
    matchLabels:
      app: redis-cluster

  template:
    metadata:
      labels:
        app: redis-cluster                # 【关联2对端】要和 selector 一致
    spec:
      containers:
      - name: redis
        image: redis:7.2
        command: ["redis-server"]
        # 【关联3】配置文件路径，要和 volumeMounts.mountPath 对应
        # mountPath=/etc/redis + ConfigMap key=redis.conf => /etc/redis/redis.conf
        args: ["/etc/redis/redis.conf"]

        ports:
        - name: client
          containerPort: 6379
        - name: gossip
          containerPort: 16379

        env:
        # 把 Pod 自身信息注入环境变量（Redis 用来告诉集群"我是谁"）
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP

        volumeMounts:
        # 【关联5左半】挂载 ConfigMap，name 对应 volumes.redis-config
        - name: redis-config
          mountPath: /etc/redis
        # 【关联6左半】挂载数据卷，name 对应 volumeClaimTemplates.redis-data
        - name: redis-data
          mountPath: /data

        readinessProbe:                   # 就绪检查：能接客了吗
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:                    # 存活检查：还活着吗
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 15
          periodSeconds: 10

      volumes:
      # 【关联5右半】声明 redis-config 卷来自哪个 ConfigMap
      - name: redis-config
        configMap:
          name: redis-cluster-config      # 【关联7】引用外部 ConfigMap 名字

  # 每个 Pod 自动创建独立 PVC（StatefulSet 的核心机制）
  volumeClaimTemplates:
  - metadata:
      name: redis-data                    # 【关联6右半】对应 volumeMounts.redis-data
    spec:
      accessModes: ["ReadWriteOnce"]
      # 【关联8】storageClassName 必须和 PV、StorageClass 一致！
      # 漏了这行 => PVC 的 storageClass 是 <unset> => 绑不上 PV
      storageClassName: local-storage
      resources:
        requests:
          storage: 1Gi
```

### 关联关系总图

```
serviceName ──► Headless Service (redis-cluster-headless)
                  └─ 给每个 Pod 发 DNS，Pod 间互通用

selector.matchLabels ◄──► template.labels  (必须一致)

args: /etc/redis/redis.conf
  ▲
  │ 路径对应
volumeMounts.redis-config ──► volumes.redis-config
  mountPath: /etc/redis           │
                                  ▼
                          configMap.name ──► ConfigMap (redis-cluster-config)

volumeMounts.redis-data ──► volumeClaimTemplates.redis-data
  mountPath: /data              │ 自动生成 PVC
                                ▼
                        PVC: redis-data-redis-cluster-N
                                │ storageClassName 匹配
                                ▼
                        PV: redis-pv-N
                                │ nodeAffinity
                                ▼
                        节点上的 /tmp/redis-data-N
```

---

## 9. 部署步骤

### 9.1 在 worker 节点上创建目录

```bash
# SSH 到 k8s2
mkdir -p /tmp/redis-data-{0,1,2}

# SSH 到 k8s3
mkdir -p /tmp/redis-data-{3,4,5}
```

### 9.2 按顺序部署

```bash
kubectl apply -f redis-sc.yaml           # 1. StorageClass
kubectl apply -f redis-pv.yaml           # 2. PV（6 个）
kubectl apply -f redis-config.yaml       # 3. ConfigMap
kubectl apply -f redis-headless.yaml     # 4. Headless Service
kubectl apply -f redis-client.yaml       # 5. Client Service（可选）
kubectl apply -f redis-statefulset.yaml  # 6. StatefulSet
```

### 9.3 验证 Pod 启动

```bash
kubectl get pods -l app=redis-cluster -w
# 应该看到 0->1->2->3->4->5 依次 Running
```

### 9.4 创建 Redis Cluster

```bash
kubectl exec -it redis-cluster-0 -- redis-cli --cluster create \
  redis-cluster-0.redis-cluster-headless:6379 \
  redis-cluster-1.redis-cluster-headless:6379 \
  redis-cluster-2.redis-cluster-headless:6379 \
  redis-cluster-3.redis-cluster-headless:6379 \
  redis-cluster-4.redis-cluster-headless:6379 \
  redis-cluster-5.redis-cluster-headless:6379 \
  --cluster-replicas 1
```

> ⚠️ 此命令**必须在 Pod 内部执行**（用 `kubectl exec`），不能在本地机器上跑，
> 因为集群内部 DNS 名只有 Pod 能解析。详见[坑 6](#坑-6dns-解析失败flannel-跨节点网络断裂)。

输入 `yes` 确认。

### 9.5 验证集群

```bash
kubectl exec -it redis-cluster-0 -- redis-cli cluster info
# cluster_state:ok

kubectl exec -it redis-cluster-0 -- redis-cli cluster nodes
# 3 主 3 从分布

kubectl exec -it redis-cluster-0 -- redis-cli -c set foo bar
kubectl exec -it redis-cluster-2 -- redis-cli -c get foo
```

---

# 第三部分：踩坑实录（真实排查全过程）

## 坑 1：PV 配置拼写错误 `cpacity`

### 现象
PV 一直创建不成功或容量为空。

### 原因
```yaml
spec:
  cpacity:        # ❌ 拼错了！
    storage: 1Gi
```
`cpacity` 应该是 `capacity`（少了个 `a`）。

### 修复
```yaml
spec:
  capacity:       # ✅ ca-pa-city
    storage: 1Gi
```

### 教训
K8s 对未知字段有时会静默忽略，`cpacity` 不被识别成"容量声明"，PV 变成没有容量的 PV，无法匹配 PVC。

---

## 坑 2：YAML 缩进错误导致解析失败

### 现象
```
error: error parsing redis-pv.yaml: error converting YAML to JSON:
yaml: line 20: mapping values are not allowed in this context
```

### 原因
YAML 对缩进极敏感。两种典型错误：

**错误 A：`required:` 缩进少了（和 `nodeAffinity:` 平级）**
```yaml
  nodeAffinity:
  required:              # ❌ 应该再缩进 2 格，现在和 nodeAffinity 平级了
    nodeSelectorTerms:
```

**错误 B：`values:` 缩进多了（比 `operator:` 深）**
```yaml
          - key: kubernetes.io/hostname
            operator: In
              values:    # ❌ 比 operator 多了 2 格，YAML 把它当成 In 的值了
              - k8s2
```

### 正确写法
```yaml
  nodeAffinity:
    required:                      # ✅ 比 nodeAffinity 多 2 格
      nodeSelectorTerms:
      - matchExpressions:          # ✅ - 和 nodeSelectorTerms 对齐
        - key: kubernetes.io/hostname
          operator: In
          values:                  # ✅ 和 operator 对齐
          - k8s2
```

### 排查方法
```bash
# 带行号查看，定位出错行
cat -n redis-pv.yaml

# 检查是否有 Tab（YAML 不允许 Tab 缩进）
grep -P '\t' redis-pv.yaml
```

### 教训
- YAML 缩进必须用空格，不能用 Tab
- `key: value` 中 value 后面不能再出现同级的 mapping（除非正确缩进）
- 报错里的 `line XX` 就是出错行号，直接看那行附近

---

## 坑 3：PVC 绑不上 PV（`unbound immediate`）

### 现象
```
Warning  FailedScheduling  pod has unbound immediate PersistentVolumeClaims. not found
```
PVC 一直 Pending，STORAGECLASS 列显示 `<unset>`。

### 原因
StatefulSet 的 `volumeClaimTemplates` 漏了 `storageClassName`：
```yaml
volumeClaimTemplates:
- metadata:
    name: redis-data
  spec:
    accessModes: ["ReadWriteOnce"]
    # ❌ 漏了 storageClassName: local-storage
    resources:
      requests:
        storage: 1Gi
```

PVC 的 storageClassName 是空的（`<unset>`），但 PV 的是 `local-storage`，**标签对不上，永远绑不上**。

而且报错里的 `immediate` 说明 PVC 走了立即绑定模式，而不是 StorageClass 配的 `WaitForFirstConsumer`--因为 PVC 根本没用这个 StorageClass。

### 修复
```yaml
volumeClaimTemplates:
- metadata:
    name: redis-data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: local-storage    # ✅ 加这一行
    resources:
      requests:
        storage: 1Gi
```

PVC 已经建错了，光改配置不够，要**删掉重建**：
```bash
kubectl delete statefulset redis-cluster
kubectl delete pvc redis-data-redis-cluster-0    # 手动删建错的 PVC
# 改完 yaml 后重新 apply
kubectl apply -f redis-statefulset.yaml
```

### 教训
- PVC 和 PV 靠 `storageClassName` 匹配，必须一致
- StatefulSet 不会自动删 PVC，改配置后要手动删旧 PVC

---

## 坑 4：`redis.conf` 找不到（ConfigMap 未绑定）

### 现象
```
Fatal error, can't open config file '/etc/redis/redis.conf': No such file or directory
```

### 原因
StatefulSet 里缺少 ConfigMap 的 volume 和 volumeMount 绑定，或者 ConfigMap 没 apply。

### 修复
确保三处绑定都在：

```yaml
# 1. volumeMounts：挂载到容器
volumeMounts:
- name: redis-config
  mountPath: /etc/redis          # ConfigMap 的 key 会变成这个目录下的文件

# 2. volumes：声明卷来自哪个 ConfigMap
volumes:
- name: redis-config
  configMap:
    name: redis-cluster-config   # 必须和 ConfigMap 名字一致

# 3. ConfigMap 必须已创建
kubectl get configmap redis-cluster-config
```

绑定链：
```
ConfigMap: redis-cluster-config (key: redis.conf)
    ↓
volumes.configMap.name = redis-cluster-config
    ↓
volumeMounts.mountPath = /etc/redis
    ↓
容器里出现: /etc/redis/redis.conf
    ↓
args: redis-server /etc/redis/redis.conf  ✅
```

---

## 坑 5：PV 卡在 Released 状态

### 现象
```
redis-pv-4   Released   default/redis-data-redis-cluster-0
```
PV 变成 Released，不能被新 PVC 自动复用，导致某个 PVC 找不到 PV。

### 原因
之前删过一次 PVC（修 storageClassName 那会儿），PVC 当时绑的是这个 PV。
因为 `reclaimPolicy: Retain`，PVC 删除后 PV 没被删，而是变成 Released，**还保留着旧 PVC 的 claimRef**。

```
PVC 删除
  ↓
PV 失去绑定的 PVC
  ↓
reclaimPolicy: Retain => PV 变 Released（不删数据，不自动复用）
  ↓
claimRef 还指向已删除的 PVC
  ↓
新 PVC 无法绑定这个 PV ❌
```

### 修复
删除 PV 的 claimRef，让它重新变 Available：
```bash
kubectl patch pv redis-pv-4 --type json -p '[{"op":"remove","path":"/spec/claimRef"}]'
```

### 教训
- `Retain` 策略保护数据但 PV 不会自动循环利用
- Released 的 PV 要手动删 `claimRef` 才能复用
- 如果从未绑上过（Available 状态），不用删

---

## 坑 6：DNS 解析失败（Flannel 跨节点网络断裂）

这是最深的坑，排查过程最长。

### 现象
```
Could not connect to Redis at redis-cluster-0.redis-cluster-headless:6379:
Temporary failure in name resolution
```

### 排查过程

**第一步：排除"在本地机器跑命令"的误区**

`redis-cli --cluster create` 必须在 Pod 内部执行（`kubectl exec`），不能在本地跑--集群内部 DNS 名只有 Pod 能解析。

**第二步：确认 Service 和 Endpoints 正常**

```bash
kubectl get svc redis-cluster-headless
# clusterIP: None ✅

kubectl get endpoints redis-cluster-headless
# 6 个 Pod IP ✅
```

Service 和 Endpoints 都正常。

**第三步：用完整域名仍失败**

```
Could not connect to Redis at redis-cluster-1...default.svc.cluster.local:6379:
Temporary failure in name resolution
```
注意错误从 `redis-cluster-0` 变成了 `redis-cluster-1`--说明 DNS 是**间歇性失败**。

**第四步：改用 IP 直连，发现 Connection timed out**

```bash
kubectl exec -it redis-cluster-0 -- redis-cli --cluster create 10.244.1.54:6379 ...
# Could not connect to Redis at 10.244.1.54:6379: Connection timed out
```

`Connection timed out`（不是 name resolution）说明：**网络层就不通**，TCP 包被丢弃。

**第五步：定位 CoreDNS 位置**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
# coredns 都在 k8s1（master），IP 10.244.0.14/15
```

CoreDNS 在 k8s1，Redis Pod 在 k8s2/k8s3。跨节点 DNS 查询到不了 CoreDNS。

**第六步：检查 Flannel**

```bash
kubectl get pods -n kube-flannel -o wide
# Flannel 在 3 个节点都 Running ✅
```

Flannel 在跑，但跨节点网络不通。

**第七步：对比 k8s2 和 k8s3 的路由（找到根因！）**

```bash
# k8s2（正常）：三条路由都有
10.244.0.0/24 via 10.244.0.0 dev flannel.1 onlink    ← 去 k8s1
10.244.1.0/24 dev cni0  src 10.244.1.1               ← 自己的
10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink    ← 去 k8s3

# k8s3（坏了）：只有自己的路由！
10.244.2.0/24 dev cni0  src 10.244.2.1
# ❌ 缺少去 k8s1 和 k8s2 的路由！
```

而且 k8s3 的 flannel.1 接口**没有 IPv4 地址**（只有 inet6）：
```
flannel.1: <UP,LOWER_UP>
    inet6 fe80::...        ← 只有 IPv6
                            ← 缺少 inet 10.244.2.0/32
```

### 根因

**k8s3 上的 Flannel 虽然显示 Running，但实际没有正确初始化**：
- flannel.1 接口没有 IPv4 地址
- 缺少跨节点路由
- 导致 k8s3 上的 Pod 既连不出去，外部也连不进来

### 修复

重启 k8s3 上的 Flannel Pod：
```bash
kubectl delete pod -n kube-flannel kube-flannel-ds-jbs8z
```

等 DaemonSet 重建后验证：
```bash
ssh k8s3 "ip addr show flannel.1 | grep inet; ip route | grep 10.244"
# 应该看到 inet 10.244.2.0/32 和三条路由
```

网络恢复后，DNS 解析正常，集群创建成功。

### 排查流程总结

```
DNS 解析失败
  ↓
确认在 Pod 内执行（不是本地）✅
  ↓
Service + Endpoints 正常 ✅
  ↓
用 IP 直连也失败（Connection timed out）→ 网络层问题
  ↓
CoreDNS 在 k8s1，Pod 在 k8s2/k8s3 → 跨节点问题
  ↓
Flannel 在跑但跨节点不通
  ↓
对比路由：k8s3 缺路由 + flannel.1 无 IPv4
  ↓
重启 k8s3 Flannel Pod → 恢复 ✅
```

### 教训
- `Temporary failure in name resolution` = 连不上 DNS 服务器（网络问题），不是域名不存在
- `Connection timed out` = TCP 包被丢弃（网络不通/防火墙）
- Pod 状态 Running 不代表网络配置正确（Flannel Pod Running 但接口没初始化）
- 排查网络问题要对比正常节点和异常节点的 `ip addr` 和 `ip route`

---

# 第四部分：关键知识点

## 16. 为什么删除 StatefulSet 不删除 PVC

### 核心原因：数据安全

```
你执行: kubectl delete statefulset mysql
          ↓
   删除 StatefulSet 对象本身
          ↓
   删除它创建的所有 Pod（Pod 有 ownerReference 指向 StatefulSet）
          ↓
   PVC 保留 ❌ 不删（PVC 没有 ownerReference）
          ↓
   PV 保留 ❌ 不删
          ↓
   磁盘上的数据保留 ❌ 不删
```

K8s 对有状态应用数据的"三重保护"：
1. 删控制器不删 PVC
2. 删 PVC（Retain 模式）不删 PV
3. 删 PV 不删磁盘数据

每一层都要**主动、显式**操作，防止"手滑丢数据"。

### Owner Reference 的区别

| 资源 | ownerReference 指向 StatefulSet？ | 删 StatefulSet 时 |
|---|---|---|
| Pod | ✅ 有 | 自动删除 |
| PVC | ❌ 没有 | 保留 |

### 彻底删除（包括数据）

```bash
kubectl delete statefulset redis-cluster       # 1. 删 StatefulSet + Pod
kubectl delete pvc -l app=redis-cluster        # 2. 手动删 PVC
kubectl delete pv redis-pv-{0..5}              # 3. 手动删 PV（Retain 模式）
# 4. 去节点上删目录（hostPath/local）
```

---

## 17. StorageClass 的本质：标签 + 行为说明书

### 标签部分
PV 和 PVC 靠 `storageClassName` 匹配，类似 label。

### 行为部分（标签做不到的）

| 字段 | 作用 |
|---|---|
| `provisioner` | 谁来造 PV（动态创建） |
| `volumeBindingMode` | 什么时候绑定 |
| `reclaimPolicy` | 删 PVC 时 PV 怎么处理 |
| `parameters` | 存储参数（磁盘类型、IOPS） |

### 静态 vs 动态

| | 静态 provisioning | 动态 provisioning |
|---|---|---|
| PV 谁建 | 管理员手动建 | StorageClass 的 provisioner 自动建 |
| provisioner | `no-provisioner` | 如 `kubernetes.io/aws-ebs` |
| StorageClass 角色 | 像"带绑定规则的标签" | 像"造盘子的配方" |

---

## 18. 概念关系总图

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes 概念层级                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Pod (最小单元)                                          │
│    ↑                                                    │
│  Deployment (无状态副本管理)                             │
│    ↑                                                    │
│  Service (稳定访问入口 + 负载均衡)                       │
│    ├── ClusterIP (普通, 有 VIP)                         │
│    └── Headless (无 VIP, 暴露 Pod IP)                   │
│         ↑                                               │
│  StatefulSet (有状态控制器)                              │
│    ├── 固定 Pod 名                                      │
│    ├── 独立 PVC                                         │
│    ├── 有序部署                                         │
│    └── 必须绑定 Headless Service                        │
│         ↑                                               │
│  Operator (完整运维自动化)                               │
│    ├── CRD (自定义资源)                                 │
│    └── Controller (调谐循环)                            │
│                                                         │
│  存储三件套：                                            │
│    PVC (声明) ←── storageClassName ──→ PV (供给)        │
│                                         ↑               │
│                                  StorageClass (行为规则) │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### K8s 里东西靠什么互相关联

| 关联类型 | 例子 | 怎么对上 |
|---|---|---|
| **资源间引用** | serviceName -> Headless Service | 名字一致 |
| **资源间引用** | configMap.name -> ConfigMap | 名字一致 |
| **资源间引用** | storageClassName -> PV / StorageClass | 名字一致 |
| **内部 name 配对** | volumeMounts.name ↔ volumes.name | 名字一致 |
| **内部 name 配对** | volumeMounts.name ↔ volumeClaimTemplates.name | 名字一致 |
| **内部 label 配对** | selector.matchLabels ↔ template.labels | label 一致 |
| **路径配对** | args 路径 ↔ mountPath + ConfigMap key | 拼起来对上 |

> **核心规律：K8s 里很多东西都是靠"名字"和"label"互相找的，名字写错一个字就对不上。**

---

## 参考资料

- [Kubernetes 官方文档 - StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes 官方文档 - Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes 官方文档 - Storage](https://kubernetes.io/docs/concepts/storage/)
- [Redis Cluster 教程](https://redis.io/docs/management/scaling/)
- [Operator 框架](https://operatorframework.io/)
- [Flannel 文档](https://github.com/flannel-io/flannel)

---

*最后更新：2026-08-03*
