### 内核参数分类

[using sysctls in k8s cluster](https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/)

- namespace 级别，可以在pod中设置，各个pod不同
    - net.core.somaxconn
    - net.ipv4.tcp_tw_reuse
    - net.ipv4.tcp_rmem,  对于 TCP 应用，这个参数会覆盖 net.core.rmem_default 和 net.core.rmem_max 的值，tcp不需要配置 发送缓冲区(wmem)

- node 级别，不能在pod中独立设置
    - net.core.rmem_max
    - net.core.wmem_max

### kubernetes 中如何配置 内核参数

- initContainer中可配置网络相关参数
```cgo
  initContainers:
  - name: setsysctl
    image: busybox
    securityContext:
      privileged: true
    command:
    - sh
    - -c
    - |
      sysctl -w net.core.somaxconn=65535
      sysctl -w net.ipv4.tcp_tw_reuse=1
      sysctl -w net.ipv4.tcp_rmem="4096 26214400 26214400"
```  

- 因ns隔离 其他ns参数需要在各自container中配置
  - node上，修改kubelet配置项，添加 --allowed-unsafe-sysctls="", 重启kubelet
  - container 定义
```cgo
   securityContext:
   sysctls:
    - name: net.core.somaxconn
      value: "4096"
    - name: net.ipv4.tcp_tw_reuse
      value: "1"
    - name: net.ipv4.tcp_rmem
      value: 4096 655350 655350
```

更多网络相关的内核参数介绍 [linux 网络内核参数](https://www.starduster.me/2020/03/02/linux-network-tuning-kernel-parameter/)


### 参考连接

- 网络优化blog: https://imroc.cc/kubernetes/best-practices/performance-optimization/network.html
- rmem 问题：https://github.com/aws/karpenter/issues/1279
- tcp_reuse: https://forum.vyos.io/t/linux-tcp-tw-reuse-2-how-is-this-set-and-what-is-the-significance/5286/2

