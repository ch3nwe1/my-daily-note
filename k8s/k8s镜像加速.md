
在`/etc/containerd/config.toml`配置文件下的`[plugins."io.containerd.grpc.v1.cri".registry.mirrors]`配置添加如下配置
```toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
	endpoint = ["https://registry.cn-hangzhou.aliyuncs.com"]
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.k8s.io"]
	endpoint = ["https://registry.aliyuncs.com/google_containers"]
```


而在2.x版本的containerd中换了另一种配置方式,在config.toml中添加如下配置
```toml
[plugins."io.containerd.cri.v1.images".registry] 
	config_path = "/etc/containerd/certs.d"
```
然后在上述目录下配置镜像加速,例如docker.io的镜像,可以创建`/etc/containerd/certs.d/docker.io/hosts.toml`文件
```toml
server = "https://registry-1.docker.io"

# 依次尝试国内大厂及高可用的加速器源
[host."https://hub-mirror.c.163.com"]
  capabilities = ["pull", "resolve"]

[host."https://mirror.baidubce.com"]
  capabilities = ["pull", "resolve"]

[host."https://docker.m.daocloud.io"]
  capabilities = ["pull", "resolve"]
```
这样就可以镜像加速了