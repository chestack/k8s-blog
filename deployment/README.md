## kubernetes 部署工具介绍
	
kubernetes的安装工具有很多，分别适用不同的场景

### All-in-one
- minikube
- hack/local-up-cluster.sh

上面两个工具是用于安装 All-in-one 环境，即所有组件都装在一个节点上。<br>
- minikube最简单，直接下载一个ISO，本地虚机启动即可访问，适合于最开始想要了解k8s的玩家。<br>
- hack/local-up-cluster.sh, 会编译源码，启动集群，适合developer单机调试k8s。


### For AWS and GKE
- kubeops, 专注于在AWS和GKE这些墙外的公有云上部署/管理kubernetes集群


### 更强大工具
作为私有云厂商，我们需要一个可以裸机上部署高可用k8s集群的工具:

- kubespray，基于ansible，在待部署机上执行各种task，包括安装：docker，etcd，flannel，以及其他需要的addons。
- kubeadm，作为k8s集群的operator，只负责k8s集群本身的生命周期管理。上面提到的k8s cluster以外的依赖组件不负责安装。


*Kubespray supports kubeadm for cluster creation since v2.3 (and deprecated non-kubeadm deployment starting from v2.8) in order to consume life cycle management domain knowledge from it and offload generic OS configuration things from it, which hopefully benefits both sides.*

[1] https://github.com/kubernetes-sigs/kubespray/blob/master/docs/comparisons.md



## 不只是部署

基于上面的介绍，kubespray集成kubeadm是社区的方向，也是我们的最佳方案。但产品化不只是部署，还要考虑其他问题:

- 升级(bug fix/大版本升级)
- 集群扩缩
- 幂等重试