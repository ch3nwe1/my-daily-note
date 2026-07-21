```yaml
apiVersion: v1
kind: Deployment
metadata:
spec:
	selector:
		matchExpressions: # 数组
			- key:
			  operator: # In|NotIn|Exists|DoesNotExist
			  values:
		matchLabels:
	template: # PodTemplateSpec
		metadata:
		spec: # Pod.md
	replicas: # 预期 Pod 的数量。这是一个指针，用于辨别显式零和未指定的值。默认为 1。
    minReadySeconds:
    strategy:
	    type: # Recreate | RollingUpdate
	    rollingUpdate:
		    maxSurge: # 在滚动更新期间，允许创建的 Pod 数量**超出**期望副本数（`replicas`）的最大值
		    maxUnavailable: # 定义了更新期间允许停机的旧 pod 的最大数量
    revisionHistory: # 保留允许回滚的旧 ReplicaSet 的数量。这是一个指针，用于辨别显式零和未指定的值。默认为 10。
    progressDeadlineSeconds:
    paused:	# 指示 Deployment 被暂停。
status:
```

## 回滚Deployment

1. 检查Deployment修改历史
```bash
kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           <none>
2           <none>
3           <none>
```
2. 查看修改历史的详细信息
```bash
kubectl rollout history deployment/nginx-deployment --revision=2
deployments "nginx-deployment" revision 2
  Labels:       app=nginx
          pod-template-hash=1159050644
  Containers:
   nginx:
    Image:      nginx:1.16.1
    Port:       80/TCP
     QoS Tier:
        cpu:      BestEffort
        memory:   BestEffort
    Environment Variables:      <none>
  No volumes.
```
3. 回滚操作
```shell
# 回滚到上一个版本
kubectl rollout undo deployment/nginx-deployment
# 回滚指定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

## 缩放Deployment (Scale)

使用`kubelet scale`命令进行pod节点扩容或者缩容
```shell
# --replicas 指定pod节点的数量
kubectl scale deploy/nginx-deployment --replicas=4
```

使用`kubectl autoscale`自动缩放pod节点
```shell
# --min 最小节点数量
# --max 最大节点数量
# --cpu-percent cpu利用率，在这个限制最大值的情况下自动缩放
kubectl autoscale deploy/nginx-deployment --min=1 --max=10 --cpu-percent=50
```

## 暂停、恢复Deployment的上线过程

暂停命令
```shell
kubectl rollout pause deployment/nginx-deployment
```
恢复命令
```shell
kubectl rollout resume deployment/nginx-deployment
```