# k8s/云原生 知识总结
	
# 技术栈

k8s平台整个技术栈包含从上面的应用到下面的硬件，是一个很大的话题。该repo聚焦k8s核心功能，即下图中的内容。

![k8s-stacks](pics/k8s-stacks.jpeg)

# 内容说明

- 本文目录对应上图，图中每一部分都是一个细分领域，每个领域都有不只一个CNCF project的开源实现。
- 本文内容包括，每一部分的基础知识、原理、领域知识，以及工作中解决的具体问题。

## 安装部署

- [`deployment tools`](cluster-lifecycle)
- [`kubespray`](cluster-lifecycle/kubespray)

## Kubernetes

kubernetes是一个庞大的项目，需要分而治之，各个击破。按照社区的方式，可以拆分为: <br>
- sub-project: controller-manager, kube-apiserver, controller-manager
- sig: sig-node, sig-network, sig-storage，sig-cluster-lifecycle

这里我们会按sub-project分类总结一些关键project的基础知识/原理，以及相关的生产问题。

- [`k8s/kubelet`](kubernetes/kubelet)
- [`k8s/controller-manager`](kubernetes/controller-manager)

## etcd
- [`etcd/存储`](etcd/storage)
- [`etcd/断网恢复重新加入集群`](etcd/rejoin)

## Operator
- [`operator deep dive`](operator.md)

## 计算
- [`docker/basic`](docker/basic)
- [`CRI调用机制`](ContainerRuntime.md)

## 网络
- [`CNI调用机制`](network/CNI.md)
- [`multus-cni/容器多网卡`](network/multiple-cni.md)
- [`kuryr-kubernetes`](network/kuryr.md)
- [`kube-ovn`](network/kube-ovn.md)
- [`kuryr-vs-kube-ovn`](network/cni-comparison.md)
- [`DNS`](network/DNS.md)
- [`ingress`](network/ingress.md)

## 存储
- [`local-volume`](storage/local-volume.md)
- [`mount-propagation`](storage/mount-propagation.md)
- [`why-bind-mount`](storage/bind-mount.md)
- [`ceph-rbd-问题排查`](storage/ceph-rbd.md)

## kata-container
- [`安全容器设计概述`](kata-container/ecr.md)
- [`云原生GPU`](kata-container/GPU.md)

## Operating System
- [`进程问题 -- D vs. Z vs. orphan`](operating-system/process.md)
- [`iowait 高`](operating-system/iowait.md)
- [`strace`](operating-system/strace.md)
- [`systemtap`](operating-system/systemtap.md)
- [`软中断`](operating-system/softirq.md)

## SRE

## golang

*********************************
# 广告位招租
*********************************