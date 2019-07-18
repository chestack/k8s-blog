# Kubernetes

kubernetes是一个庞大的项目，需要分而治之，各个击破。按照社区的方式，可以拆分为: <br>
- sub-project: controller-manager, apiserver, controller-manager
- sig: sig-node, sig-network, sig-storage

这里我们会总结一些关键sub-project的基础知识/原理，所以会为sub-project创建目录。也会总结具体的问题。

- [`kubelet启动`](kubelet/startup.md)
- [`kubelet/Pods工作原理`](kubelet/pods.md)
- [`kubelet/PVC工作原理`](kubelet/pvc.md)

- [`调整内存/kubepods memory.limits < memory.usage`](kubelet/memory.md)
- [`pods状态与containers状态不同步`](kubelet/pods-containers.md)
- [`PVC无法挂载`](kubelet/pvc-mount.md)