**Pod** 是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元

## pod定义
```yaml
apiVersion: v1
kind: Pod
metadata: # 元数据
	name: #资源唯一名称
	generateName: # 自动生成 Pod 名 适用于 Job / 临时 Pod 指定前缀
	namespace: #资源所属命名空间,默认是default
	labels: # 可用于组织和分类（确定范围和选择）对象的字符串键和值的映射
	annotations: # 存储非结构化扩展信息 - - 不参与 label selector
spec:
	containers: # 容器列表
	initContainers: # 属于 Pod 的 Init 容器列表
	ephemeralContainers: # 在此 Pod 中运行的临时容器列表
	imagePullSecrets: # 用户拉取私有镜像的凭据
	enableServiceLinks: # boolean  enableServiceLinks 指示是否应将有关服务的信息注入到 Pod 的环境变量中
	os: # 指定 Pod 中容器的操作系统。如果设置了此属性，则某些 Pod 和容器字段会受到限制
		name: # name 是操作系统的名称。当前支持的值是 `linux` 和 `windows`
	
```