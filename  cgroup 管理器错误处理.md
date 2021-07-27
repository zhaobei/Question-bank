

# 错误处理



## 现象

问题出现情况：在麒麟ARM芯片的机器上搭建k8s，其中的的一个组件cordons 发现启动失败，查看日志如下所示：

```shell
Error: failed to start container "coredns": Error response from daemon:
OCI runtime create failed: container_linux.go:318: starting container process caused "process_linux.go:264: applying cgroup configuration for process caused \"No such device or address\"": unknown
```



然后查看kubelet的状态（System status kubelet）,失败了，再查看kubelet的日志（journalctl -xe | grep kubelet）

```shell
skipping: failed to "StartContainer" for "coredns" with RunContainerError: "failed to start container \"b126bf6a717cc76379b15e8540a73d0902bae516373eaa4276411345106a2c7b\": Error response from daemon: OCI runtime create failed: container_linux.go:318: starting container process caused \"process_linux.go:264: applying cgroup configuration for process caused \\\"No such device or address\\\"\": unknown"


```





## 问题分析

从错误原因来看是 cgroup 配置导致“没有这样的设备或地址”，首先检查 docker 和 kubelet 的管理器配置:

```shell
# 检查 docker
cat /etc/docker/daemon.json | grep cgroupdriver
-->"exec-opts": ["native.cgroupdriver=cgroupfs"],



# 检查 kubelet
cat /var/lib/kubelet/config.yaml | grep cgroupDriver
-->cgroupDriver: cgroupfs

```



发现 docker 为 cgroup, 而 kubelet 为 systemd,。二者不一样，cgroupfs与 systemd 一起使用意味着将有两个不同的 cgroup 管理器。如果 kubelet 使用一个 cgroup 驱动程序的语义创建了 Pod，那么在尝试为此类现有 Pod 重新创建 Pod 沙箱时，将容器runtime更改为另一个 cgroup 驱动程序可能会导致此类错误。





## 解决方案

以下是通过显式配置两者来确保 docker 和 kubelet 使用相同的 cgroups 驱动程序的方法：

docker：

```shell
cat /etc/docker/daemon.json

{
    “ exec-opts ”：[ “ native.cgroupdriver=cgroupfs ” ]
}
```



```shell
Kubelet（使用 with 设置时kubeadm）：

cat cat /var/lib/kubelet/config.yaml
->cgroupDriver: cgroupfs
```







kubelet 配置 cgrop. 

https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/

https://gist.github.com/MOZGIII/22bf4eb811ff5d4e0bbe36444422b6d3

https://github.com/cri-o/cri-o/issues/832