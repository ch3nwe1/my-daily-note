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
			limits: # 资源请求的上线
				cpu: # 占用系统cpu的资源，以时间进行切分，1个cpu核心等于1000毫核
				memory: # 占用的最大硬盘内存，使用2进制计算单位，Mi,Gi,Ki
			requests: # 资源请求的下限
				cpu:
				memory:
		 resizePolicy: # 调整resources配置的资源时，pod重启策略
			 - resourceName: # cpu和memory
			   restartPolicy: # NotRequired 不需要重启，RestartContainer需要重启
		 lifecycle:
			 postStart: # 创建容器后立即调用 postStart。如果处理程序失败，则容器将根据其重新启动策略终止并重新启动。 容器的其他管理阻塞直到钩子完成
				 exec: 
					 command:  # command 是要在容器内执行的命令行，命令的工作目录是容器文件系统中的根目录（'/'）
				 httpGet: 
					 port:
					 host:
					 httpHeaders:
						 - name:
						   value:
					 path:
					 scheme:
				 tcpSocket: # 已经弃用
					 port:
					 host:
			 preStop: # 与 postStart一致
		 terminationMessagePath:
		 terminationMessagePolicy:
		 livenessProbe: # 定期探针容器活跃度。如果探针失败，容器将重新启动。无法更新
		 readinessProbe: # 定期探测容器服务就绪情况。如果探针失败，容器将被从服务端点中删除。
		 startupProbe: # startupProbe 表示 Pod 已成功初始化。如果设置了此字段，则此探针成功完成之前不会执行其他探针。 如果这个探针失败，Pod 会重新启动，就像存活态探针失败一样。 这可用于在 Pod 生命周期开始时提供不同的探针参数，此时加载数据或预热缓存可能需要比稳态操作期间更长的时间
			 exec:
				 command:
			 httpGet:
				  port :
				  host:
				  httpHeaders:
				  path:
				  schema:
			 initialDelaySeconds: # 容器启动后延迟多久开始调用liveness probes 
			 terminationGracePeriodSeconds:
			 periodSeconds:
			 timeoutSeconds:
			 failureThreshold:
			 successThreshold:   
			 grpc:
				 port:
				 service: 
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
		  emptyDir: #代表临时目录，使用{},多用于一个pod中多容器之间的交互，与pod生命周期相同,默认路径/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~empty-dir/<volume-name>/
		  hostPath:
			  path:
			  type:
	nodeSelector: # nodeSelector 是一个选择算符，这些算符必须取值为 true 才能认为 Pod 适合在节点上运行。 选择算符必须与节点的标签匹配，以便在该节点上调度 Pod
	nodeName:  # nodeName 是将此 Pod 调度到特定节点的请求。 如果字段值不为空，调度器只是直接将这个 Pod 调度到所指定节点上，假设节点符合资源要求。
	affinity: # 亲和度
	tolerations: # 容忍度
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

### volumes
#### hostPath

#####  type

| Directory         | 必须是目录      |
| ----------------- | ---------- |
| DirectoryOrCreate | 不存在就创建     |
| File              | 必须是文件      |
| FileOrCreate      | 不存在就创建     |
| Socket            | 必须是 socket |
| CharDevice        | 字符设备       |
| BlockDevice       | 块设备        |

## pod 生命周期

### pod 阶段
pod的`status`字段是个[PodStatus]()对象，其中包含一个`phase`字段.

- Pending: Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间。
- Running:  Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。
- Succeeded: Pod 中的所有容器都已成功结束，并且不会再重启。
- Failed: Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止，且未被设置为自动重启。
- Unknown: 因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。
### 容器状态
- Waiting: 容器仍在运行前的准备阶段（如正在拉取镜像、等待依赖卷挂载）。
- Running: 容器正在无间断地正常执行。
- Terminated: 容器已经开始执行并已运行完毕（或由于某种原因失败退出）。
### pod condition
- PodScheduled: Pod 已经被调度到某节点；
- PodReadyToStartContainers: Pod 沙箱被成功创建并且配置了网络
- ContainersReady: Pod 中所有容器都已就绪；
- Initialized: 所有的 [Init 容器](https://v1-35.docs.kubernetes.io/zh-cn/docs/concepts/workloads/pods/init-containers/)都已成功完成；
- Ready: Pod 可以为请求提供服务，并且应该被添加到对应服务的负载均衡池中。
- `DisruptionTarget`：由于干扰（例如抢占、驱逐或垃圾回收），Pod 即将被终止。
- PodResizePending：已请求对 Pod 进行调整大小，但尚无法应用。 详见 Pod 调整大小状态。
- PodResizeInProgress：Pod 正在调整大小中。 详见 Pod 调整大小状态。