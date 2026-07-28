# StatefulSet 与有状态工作负载

> 本文总结自一次关于 K8s 有状态工作负载的学习会话，从 Service 基础到 StatefulSet、Headless Service、Operator，最后给出了一个完整的 Redis Cluster 实战配置。

---

## 目录

1. [Service 基础概念](#1-service-基础概念)
2. [Headless Service（无头服务）](#2-headless-service无头服务)
3. [StatefulSet（有状态控制器）](#3-statefulset有状态控制器)
4. [为什么 StatefulSet 必须用 Headless Service](#4-为什么-statefulset-必须用-headless-service)
5. [Operator 模式](#5-operator-模式)
6. [实战：Redis Cluster StatefulSet 配置](#6-实战redis-cluster-statefulset-配置)

---

## 1. Service 基础概念

### 1.1 Service 是什么

**Service = 稳定的"入口" + 自动的"分发规则"**

解决的问题：
- Pod IP 随机变化，无法稳定访问
- 多个 Pod 之间如何负载均衡
- Pod 挂了如何感知并剔除
- 扩缩容后如何自动更新后端列表

### 1.2 Service 的三种核心行为

1. **提供不变的访问地址**（DNS 名 + 虚拟 IP）
2. **自动追踪后端哪些 Pod 是活的**（通过 label selector）
3. **把请求分发到健康的 Pod 上**（负载均衡）

### 1.3 简单示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-web-svc
spec:
  selector:                # 找出所有 label 是 app=my-web 的 Pod
    app: my-web
  ports:
  - port: 80               # Service 对外暴露的端口
    targetPort: 8080       # Pod 里容器监听的端口
```

### 1.4 VIP（Virtual IP）

**VIP = 虚拟 IP**，不真实存在于任何网卡上，只是 kube-proxy 设置的 iptables 规则中的"钩子"。

类比：公司的 400 客服电话 → 不对应具体电话机，自动转接到值班员工。

访问 VIP 的流程：
```
请求 → iptables 规则拦截（"这是 Service 的 VIP"）
     → 负载均衡算法选一个后端 Pod
     → 转发到 Pod 真实 IP
```

### 1.5 Service 类型

| 类型 | 说明 | 适用场景 |
|---|---|---|
| **ClusterIP** | 默认，只能在集群内部访问 | 服务间互相调用 |
| **LoadBalancer** | 云厂商分配公网 IP/ELB | 对外暴露 Web 应用 |
| NodePort | 在每个节点上暴露固定端口 | 开发测试 |
| ExternalName | 外部 DNS 的 CNAME | 连接外部服务 |

### 1.6 心智模型：公司组织架构

- **Pod** = 具体的员工（会离职、入职）
- **Deployment** = 某个部门（"我要 5 个客服"——负责招人/裁人）
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
  client → VIP → kube-proxy → Pod A / Pod B / Pod C (负载均衡)

Headless Service:
  client → DNS 查询 → [Pod A IP, Pod B IP, Pod C IP] (客户端自己选)
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

### 2.4 DNS 行为

```bash
# 主域名解析：返回所有 Pod IP
$ nslookup my-headless-svc
Name: my-headless-svc → 10.0.0.5
Name: my-headless-svc → 10.0.0.6
Name: my-headless-svc → 10.0.0.7
```

### 2.5 使用场景

| 场景 | 为什么需要 Headless |
|---|---|
| StatefulSet 节点互访 | 每个 Pod 需要稳定、可预测的 DNS |
| 客户端自己做负载均衡 | gRPC 长连接、数据库连接池 |
| 服务发现 | 客户端要枚举"目前有哪些健康实例" |

---

## 3. StatefulSet（有状态控制器）

### 3.1 定义

**StatefulSet = 给 Pod 提供"稳定身份 + 独立存储 + 有序部署"的控制器**

与 Deployment 对比：

| 特性 | Deployment | StatefulSet |
|---|---|---|
| Pod 名 | 随机（`app-5f8b-x2k9p`） | 固定序号（`mysql-0`, `mysql-1`） |
| 启停顺序 | 无序 | 有序（0→1→2 启动，2→1→0 停止） |
| 存储 | 共享 PVC，Pod 重建后丢失 | 每个 Pod 独享 PVC，重建后仍绑定同一份数据 |
| 网络身份 | 共享 Service VIP | 每个 Pod 独立 DNS |
| 适用场景 | Web 服务、API | 数据库、消息队列 |

### 3.2 配置示例

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless   # 👈 必须绑定 Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:         # 👈 每个 Pod 自动创建独立 PVC
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 3.3 每个 Pod 会得到什么

```
mysql-0  →  PVC: data-mysql-0  →  DNS: mysql-0.mysql-headless.ns.svc.cluster.local
mysql-1  →  PVC: data-mysql-1  →  DNS: mysql-1.mysql-headless.ns.svc.cluster.local
mysql-2  →  PVC: data-mysql-2  →  DNS: mysql-2.mysql-headless.ns.svc.cluster.local
```

### 3.4 关键特点

- **稳定的网络标识**：Pod 重启后名字、DNS 不变
- **稳定的存储**：Pod 重建后还绑着同一份 PVC
- **有序部署/扩缩/删除**：严格按序号操作
- **适合有主从/分片概念的应用**：MySQL 主从、Kafka broker、Redis Cluster

---

## 4. 为什么 StatefulSet 必须用 Headless Service

### 4.1 根本原因

普通 ClusterIP Service 的 VIP **抹掉了 Pod 身份**——客户端无法指名道姓地访问某个具体 Pod。

但对有状态应用（MySQL 主从、Kafka、Raft 集群），客户端**必须知道数据在哪台**，负载均衡反而是在帮倒忙。

### 4.2 Headless Service 解决了什么

1. **给每个 Pod 发独立 DNS**：`<pod-name>.<svc-name>.<ns>.svc.cluster.local`
2. **主域名解析返回所有 Pod IP**：客户端自己决定连谁

### 4.3 StatefulSet 控制器怎么用

StatefulSet 控制器为每个 Pod 注入 `hostname` 和 `subdomain` 字段：

```yaml
spec:
  hostname: mysql-0              # 来自 Pod 名
  subdomain: mysql               # 来自 serviceName
```

kube-dns 看到这两个字段 + Headless Service，生成稳定 DNS。

### 4.4 如果不用 Headless 会怎样

- ❌ `mysql-0.mysql-svc.default.svc.cluster.local` 这个 DNS **不存在**
- ❌ Pod 间无法用稳定名字互相访问
- ❌ 有状态协议（主从选举、分片路由、Raft 投票）全部失效
- ❌ StatefulSet 失去"稳定网络身份"承诺

### 4.5 因果链

```
StatefulSet (serviceName=mysql)
  ↓
控制器为每个 Pod 注入 hostname=mysql-N, subdomain=mysql
  ↓
Pod 创建（带 hostname/subdomain）
  ↓
kube-dns 检查 subdomain 对应的 Service
  ├── Service 有 clusterIP → DNS 只返回 VIP ❌ (Pod 身份丢失)
  └── Service 是 headless  → DNS 返回 <pod>.<svc>.<ns>.svc... ✅
```

---

## 5. Operator 模式

### 5.1 定义

**Operator = 一段自定义的控制逻辑 + 一组自定义资源（CRD）**

- 自动运维复杂应用（MySQL、Kafka、ES 等）
- 把运维知识编码成软件
- StatefulSet 的"升级版"

### 5.2 为什么需要 Operator

StatefulSet 只解决了"起 Pod + 绑存储"，但真正的运维还需要：

| 场景 | StatefulSet 能做 | Operator 做 |
|---|---|---|
| 主库挂了 | 重启 Pod | 自动提升从库为主库、改路由 |
| 扩容 | 起新 Pod | 自动配置复制、同步数据 |
| 备份 | ❌ | 定时 dump 到 S3 |
| 滚动升级 | 重启 Pod | 按顺序、切流量、检查兼容性 |
| 读写分离 | ❌ | 自动部署 Router 层 |

### 5.3 核心机制：Reconciliation Loop（调谐循环）

```
读取用户写的"期望状态" (CR YAML)
  ↓
读取集群里的"实际状态" (Pod/Service/PVC/...)
  ↓
对比：期望 vs 实际
  ├── 一致 → 等待
  └── 不一致 → 执行动作让实际=期望
                  ↓
               再次对比（持续循环）
```

### 5.4 CRD + Controller 架构

```yaml
# CRD：自定义资源类型
apiVersion: mysql.example.com/v1
kind: MySQLCluster        # 👈 新资源类型
metadata:
  name: my-cluster
spec:
  replicas: 3
  version: "8.0"
  backup:
    enabled: true
```

Controller 监听 CRD 变化 → 自动创建/修改底层资源（StatefulSet、Service、CronJob 等）

### 5.5 常见 Operator

| Operator | 管什么 | 自动化什么 |
|---|---|---|
| **Strimzi** | Kafka | Topic、Partition、滚动升级 |
| **Prometheus Operator** | Prometheus | 自动生成监控配置 |
| **Cert-Manager** | TLS 证书 | 自动申请、续期 |
| **ArgoCD** | GitOps 部署 | 监听 Git 自动同步 |
| **Elasticsearch Operator** | ES 集群 | 分片、扩容、索引管理 |

### 5.6 概念层级

| 工具 | 解决的问题 |
|---|---|
| Pod | 单个容器怎么跑 |
| Deployment | 无状态 Pod 副本管理 |
| Service | 怎么稳定访问 Pod |
| StatefulSet | 有状态 Pod 管理（固定名、独立存储） |
| Headless Service | 有状态 Pod 互相发现 |
| **Operator** | **复杂应用的完整运维** |

---

## 6. 实战：Redis Cluster StatefulSet 配置

### 6.1 架构

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

最少需要 **3 主 + 3 从 = 6 个 Pod**。

### 6.2 ConfigMap：redis-config.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster-config
data:
  redis.conf: |
    cluster-enabled yes
    cluster-config-file nodes.conf
    cluster-node-timeout 5000
    appendonly yes
    cluster-announce-port 6379
    cluster-announce-bus-port 16379
    bind 0.0.0.0
    protected-mode no
```

### 6.3 Headless Service：redis-headless.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster-headless
spec:
  clusterIP: None                    # 👈 无头 Service
  selector:
    app: redis-cluster
  ports:
  - name: client
    port: 6379
    targetPort: 6379
  - name: gossip
    port: 16379
    targetPort: 16379
```

### 6.4 普通 Service（客户端入口）：redis-client.yaml

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

### 6.5 StatefulSet：redis-statefulset.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster-headless     # 👈 绑定 headless service
  replicas: 6                             # 3 主 + 3 从
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:7.2
        command: ["redis-server"]
        args: ["/etc/redis/redis.conf"]
        ports:
        - name: client
          containerPort: 6379
        - name: gossip
          containerPort: 16379
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: redis-config
          mountPath: /etc/redis
        - name: data
          mountPath: /data
        readinessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 15
          periodSeconds: 10
      volumes:
      - name: redis-config
        configMap:
          name: redis-cluster-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

### 6.6 部署步骤

```bash
# 1. 部署所有资源
kubectl apply -f redis-config.yaml
kubectl apply -f redis-headless.yaml
kubectl apply -f redis-client.yaml
kubectl apply -f redis-statefulset.yaml

# 2. 观察 Pod 启动（有序：0→1→2→3→4→5）
kubectl get pods -l app=redis-cluster -w

# 3. 验证 Headless Service DNS
kubectl exec redis-cluster-0 -- nslookup redis-cluster-headless

# 4. 创建 Redis Cluster（关键一步！）
PODS="redis-cluster-0.redis-cluster-headless:6379 \
      redis-cluster-1.redis-cluster-headless:6379 \
      redis-cluster-2.redis-cluster-headless:6379 \
      redis-cluster-3.redis-cluster-headless:6379 \
      redis-cluster-4.redis-cluster-headless:6379 \
      redis-cluster-5.redis-cluster-headless:6379"

kubectl exec -it redis-cluster-0 -- \
  redis-cli --cluster create $PODS \
  --cluster-replicas 1

# 5. 验证
kubectl exec -it redis-cluster-0 -- redis-cli cluster info
kubectl exec -it redis-cluster-0 -- redis-cli cluster nodes

# 6. 测试读写
kubectl exec -it redis-cluster-0 -- redis-cli -c set foo bar
kubectl exec -it redis-cluster-2 -- redis-cli -c get foo
```

### 6.7 可以观察的现象

**现象 1：杀掉主库，自动故障转移**
```bash
kubectl exec redis-cluster-0 -- redis-cli cluster nodes | grep master
kubectl delete pod redis-cluster-0
# 等重启后，原本的 replica 变成了 master
```

**现象 2：Pod 重建后数据还在**
```bash
kubectl exec -it redis-cluster-0 -- redis-cli -c set test-key "hello"
kubectl delete pod redis-cluster-0
# Pod 重建后数据还在
kubectl exec -it redis-cluster-0 -- redis-cli -c get test-key
```

**现象 3：扩容**
```bash
kubectl scale statefulset redis-cluster --replicas=8
# 新节点需要手动加入集群并重新分片（这就是为什么生产环境要用 Operator）
```

### 6.8 清理

```bash
kubectl delete -f redis-statefulset.yaml
kubectl delete -f redis-client.yaml
kubectl delete -f redis-headless.yaml
kubectl delete -f redis-config.yaml
# StatefulSet 不会自动删 PVC，要手动清
kubectl delete pvc -l app=redis-cluster
```

---

## 附录：概念关系图

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
└─────────────────────────────────────────────────────────┘
```

---

## 参考资料

- [Kubernetes 官方文档 - StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes 官方文档 - Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Redis Cluster 教程](https://redis.io/docs/management/scaling/)
- [Operator 框架](https://operatorframework.io/)

---

*最后更新: 2026-07-29*
