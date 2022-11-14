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
作为私有云厂商，我们需要一个可以裸机/虚机上部署高可用k8s集群的工具:

- kubespray，基于ansible，在待部署机上执行各种task，包括安装：containerd，etcd，flannel，以及其他需要的addons。
- clusterAPI, k8s社区主推部署工具，声明式YAML定义cluster 配置，向下可以部署在裸金属，OpenStack 及其他public cloud上。
- kubeadm，作为k8s集群的operator，只负责k8s集群本身的生命周期管理。上面提到的k8s cluster以外的依赖组件不负责安装。

- clusterAPI 和 kubespray 都是负责集群完整部署的工具，对于k8s核心组件部分(k8s+etcd) 都依赖于kubeadm
- kubespray的优势就是 灵活，因为是写ansible task，所以可以很方便的定义任何动作，只要代码写得好 所有需求都应付得了, 所以产品中 目前主要使用kubespray
- clusterAPI, 优势在多集群管理，尤其是部署在不同cloud provider的场景，因为是operator模式，有些失败重试逻辑的封装但失去了一些灵活性，想改逻辑就要改代码，产品中目前尝试作为EKS集群的部署工具，替换magnum.

## 不只是部署

基于上面的介绍，kubespray集成kubeadm是社区的方向，也是我们的最佳方案。但产品化不只是部署，还要考虑其他问题:

- 升级(bug fix/大版本升级)
- 集群扩缩
- 重试幂等