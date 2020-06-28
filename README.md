# k8s/container 知识总结
	
# 技术栈

如下图，云平台整个技术栈包含从上面的应用到下面的硬件，是一个很大的话题。该Repo聚焦在k8s平台核心功能，即图中绿色矩形框里的内容。

![k8s-stacks](pics/k8s-stacks.jpg) 

# 掌握程度
对一个技术的掌握程度，我自己划分为四个阶段:
- 概念，维基百科看过介绍，理解这个技术的思想是什么，解决什么问题。
- 原理，认真读过官方文档，了解主要的配置参数，了解其向下依赖的技术是什么。这个阶段，出了问题会很快有思路
- 代码，看过源码，知道分几个模块，模块间交互方式，了解常见的问题。这个阶段，根据log 可以快递定位并fix问题
- owner，这东西我自己写的，了然于胸。这个阶段，好的作品会继续维护，不好的只想换坑，哈哈

# 内容目录

绿色框内其实已经包含了大量内容，其中每个方框都是一个知识领域，都需要大量的技术积累。
- 每个领域又可以细分成不同的功能模块/组件
- 每个功能模块又包含了很多开源实现

这里会结合工作中遇到的实际问题，以问题为着力点，记录相关的基础知识以及问题的处理过程。


## 安装部署

- [`deployment tools`](deployment)
- [`kubespray`](deployment/kubespray)

## Kubernetes

kubernetes是一个庞大的项目，需要分而治之，各个击破。按照社区的方式，可以拆分为: <br>
- sub-project: controller-manager, apiserver, controller-manager
- sig: sig-node, sig-network, sig-storage，sig-cluster-lifecycle

这里我们会按sub-project分类总结一些关键project的基础知识/原理，以及相关的生产问题。

- [`k8s/kubelet`](kubernetes/kubelet)
- [`k8s/controller-manager`](kubernetes/controller-manager)
- [`k8s/apiserver`]
- [`k8s/debug`]


## Container Runtime

- [`梳理Container Runtime调用机制`](ContainerRuntime.md)
- [`梳理CNI调用机制`](CNI.md)

## etcd

- [`etcd/basic`](etcd/basic)
- [`etcd/断网恢复重新加入集群`](etcd/region)

## Docker

- [`docker/basic`](docker/basic)


## network
- [`net/multiple-cni`](network/multiple-cni.md)


## Ceph


## Operating System

- [`os/shell`]
- [`os/排查系统问题`]