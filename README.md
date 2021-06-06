# k8s/云原生 知识总结
	
# 技术栈

k8s平台整个技术栈包含从上面的应用到下面的硬件，是一个很大的话题。该Repo结合工作内容聚焦k8s核心功能，即下图中的内容。

![k8s-stacks](pics/k8s-stacks.jpeg)

# 内容目录

- 上图中每一部分都是一个细分领域，每个领域都有不只一个CNCF project的开源实现，每个领域都需要一定的领域知识。
- 本文内容会结合工作中遇到的实际问题，以具体问题为抓手，记录每一部分的基础知识以及问题的处理过程。

## 安装部署

- [`deployment tools`](deployment)
- [`kubespray`](deployment/kubespray)

## Kubernetes

kubernetes是一个庞大的项目，需要分而治之，各个击破。按照社区的方式，可以拆分为: <br>
- sub-project: controller-manager, kube-apiserver, controller-manager
- sig: sig-node, sig-network, sig-storage，sig-cluster-lifecycle

这里我们会按sub-project分类总结一些关键project的基础知识/原理，以及相关的生产问题。

- [`k8s/kubelet`](kubernetes/kubelet)
- [`k8s/controller-manager`](kubernetes/controller-manager)
- [`k8s/apiserver`]
- [`k8s/debug`]

## Operator

- [`operator deep dive`](operator.md)

## 计算

- [`docker/basic`](docker/basic)
- [`CRI调用机制`](ContainerRuntime.md)
- [`云原生GPU`](GPU.md)
- [`安全容器设计概述`](ecr.md)

## 网络
- [`CNI调用机制`](network/CNI.md)
- [`multus-cni/容器多网卡`](network/multiple-cni.md)
- [`kuryr-kubernetes`](network/kuryr.md)
- [`kube-ovn`](network/kube-ovn.md)
- [`kuryr-vs-kube-ovn`](network/kuryr-vs-kube-ovn.md)
- [`DNS`](network/DNS.md)
- [`ingress`](network/ingress.md)

## 存储


## Operating System


## etcd

- [`etcd/basic`](etcd/basic)
- [`etcd/断网恢复重新加入集群`](etcd/region)


# 掌握程度
对一个技术的掌握程度，我自己划分为四个阶段:
- 概念，维基百科看过介绍，理解这个技术的思想是什么，解决什么问题。
- 原理，认真读过官方文档，了解主要的配置参数，了解其向下依赖的技术是什么。这个阶段，出了问题会很快有思路
- 代码，看过源码，知道分几个模块，模块间交互方式，了解常见的问题。这个阶段，根据log 可以快递定位并fix问题
- owner，这东西我自己写的，了然于胸。这个阶段，好的作品会继续维护，不好的只想换坑，哈哈