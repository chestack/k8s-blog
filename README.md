# k8s/container 知识总结
	
# 技术栈

如下图，云平台整个技术栈包含从上面的应用到下面的硬件，是一个很大的话题。该Repo聚焦在k8s平台核心功能，即图中绿色矩形框里的内容。

![k8s-stacks](pics/k8s-stacks.jpg) 

# 内容目录

绿色框内其实已经包含了大量内容，其中每个方框都是一个知识领域，都需要大量的技术积累。
- 每个领域又可以细分成不同的功能模块/组件
- 每个功能模块又包含了很多开源实现

这里会结合工作中遇到的实际问题，以问题为着力点，记录相关的基础知识以及问题的处理过程。

## Kubernetes

- [`k8s/kubelet`](kubernetes/kubelet)
- [`k8s/controller-manager`](kubernetes/controller-manager)

## etcd


## Docker

- [`docker/rootfs`](docker/basic)