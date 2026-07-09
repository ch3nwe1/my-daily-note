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
		- name: # 容器名称 有规范，遵循DNS_LABEL
		  image: # 镜像名称
		  imagePullPolicy: # 镜像拉取策略 Always,Never,IfNotPresent
		  # EntryPoint 覆盖容器中的启动命令
		  command:
		  args: 
		  workingDir:
		  ports:
			 containerPort:
			 hostIP: # 宿主机IP
			 hostPort: # 宿主机端口
			 name:
			 protocol:
		 env: # 存入容器中的环境变量
			 - name:
			   value: # 可以使用${}引用其他环境变量
			   valueFrom:
				   configMapKeyRef:
					   key:
					   name:
					   optional:
					fieldRef:
						fieldPath:
						apiVersion: # 默认v1
					resourceFieldRef:
					secretKeyRef:
						key:
						name:
						optional:
		 envFrom: # 将configmap或secret中的所有数据都加入到环境变量中
			configMapRef: # 引用configmap
				name: # configmap的名字
				optional: # 引用的资源不存在是否报错 true代表可选的，不报错
			prefix:
			secretRef:
				name:
				optional:
		 volumeMounts:
			- :
			  name: # 数据卷的名称，与volumes中的name对应
			  mountPropagation:
			  mountPath: # 挂载到容器中的路径
			  readOnly: # 容器只能读取数据，不能写入
			  subPath: # 针对数据卷的子路径或者文件
			  subPathExpr:
		 volumeDevices:
			- devicePath:
			  name:
		 resources:
			claims:
				- name:
			limits:
				cpu:
				memory:
			requests:
				cpu:
				memory:
		 resizePolicy: 
			 - resourceName:
			   restartPolicy:
		 lifecycle:
			 postStart:
			 preStop:
		 terminationMessagePath:
		 terminationMessagePolicy:
		 livenessProbe:
		 readinessProbe:
		 startupProbe:
		 restartPolicy:
		 securityContext:
			 runAsUser:
			 runAsNonRoot:
			 runAsGroup:
			 readOnlyRootFilesystem:
			 procMount:
			 privileged:
			 allowPrivilegeEscalation:
			 capabilities:
		 stdin:
		 stdinOnce:
		 tty:	 		       	           
	initContainers: # 属于 Pod 的 Init 容器列表
	ephemeralContainers: # 在此 Pod 中运行的临时容器列表
	imagePullSecrets: # 用户拉取私有镜像的凭据
	enableServiceLinks: # boolean  enableServiceLinks 指示是否应将有关服务的信息注入到 Pod 的环境变量中
	os: # 指定 Pod 中容器的操作系统。如果设置了此属性，则某些 Pod 和容器字段会受到限制
		name: # name 是操作系统的名称。当前支持的值是 `linux` 和 `windows`
	volumes: # 可以由属于 Pod 的容器挂载的卷列表
		- name: # 数据卷唯一标识
		  emptyDir: #代表临时目录，使用{},多用于一个pod中多容器之间的交互，与pod生命周期相同
		  hostPath:
			  path:
			  type:
	nodeSelector: # nodeSelector 是一个选择算符，这些算符必须取值为 true 才能认为 Pod 适合在节点上运行。 选择算符必须与节点的标签匹配，以便在该节点上调度 Pod
	nodeName:  # nodeName 是将此 Pod 调度到特定节点的请求。 如果字段值不为空，调度器只是直接将这个 Pod 调度到所指定节点上，假设节点符合资源要求。
	affinity: # 亲和度
	tolerations：# 容忍度
	schedulerName： #- 如果设置了此字段，则 Pod 将由指定的调度器调度。如果未指定，则使用默认调度器来调度 Pod。
	runtimeClassName: 
	priorityClassName: 
	priority:
	preemptionPolicy: 
	topologySpreadConstraints:
	overhead:
	# ----------------生命周期------------------------
	restartPolicy: # `Always`、`OnFailure`、`Never`
	terminationGracePeriodSeconds:  # 可选字段，表示 Pod 需要体面终止的所需的时长
	activeDeadlineSeconds:  # 在系统将主动尝试将此 Pod 标记为已失败并杀死相关容器之前，Pod 可能在节点上活跃的时长； 时长计算基于 startTime 计算（以秒为单位）。字段值必须是正整数
	readinessGate: #
	# --------------------网络------------------------ 
	hostname:
	setHostnameAsFQDN:
	subdomain:
	hostAliases:
	dnsConfig:
	dnsPolicy:
	hostNetwork:
	hostPID:
	hostIPC:
	shareProcessNamespace:
	serviceAccountName:
	automountServiceAccountToken:
	securityContext:
	            
```

### containers.imagePullPolicy
- Always: 总是拉取镜像
- IfNotPresent：如果镜像不存在，才获取镜像
- Never: 从不获取镜像，使用本地镜像，如果本地不存在，pod启动报错
如果镜像标签中是`latest`,则默认值是`Always`,否则是`IfNotPresent`。