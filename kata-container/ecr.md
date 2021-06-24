## 安全容器架构设计

- 安全容器选型
- 安装部署方案
- 对EOS平面的影响
- 资源管理方案
- 网络方案
- 存储方案
- 安全容器使用GPU


### 安全容器技术选型

```kubelet --> containerd --> ecr (easystack container runtime)```

- 参照社区kata-container 
  * 版本: v1.2.0
  
- arm memory hotplug
  * qemu-5.1.0, qemu-kvm-ecr
  * guest kernel 4.18.0-240(加补丁), host kernel 4.18.0-147
  

### 安装部署 -- 安全容器云产品

- 节点安装
  * License 定义节点角色
  * EOS 安装/扩容

- 云产品依赖
  * 依赖Foundation已安装qemu-kvm-ecr
  * 依赖keystone, neutron, neutron LB


### 对EOS平面的影响

- 调度
  * 独立节点运行安全容器Pods，通过taints防止非安全容器Pods调度到安全容器节点

- 网络
  * 控制平面使用flannel网络，安全容器使用neutron网络


### 资源(内存 CPU)管理方案

- 节点
  * 内存: 总内存-3G
  * cpu: 不限制
  
- 容器
  * Pod Spec和普通Pod一致
  * Guest VM 默认启动资源 1C 512M
  * 容器资源通过hotplug机制在虚机中插拔
     - x86粒度: cpu整数个，内存128M
     - arm粒度: cpu整数个，内存1G
  * Host上计算Pod资源使用：Pod资源 + 虚机qemu进程组资源(cpu 50m，memory 300M)
 
 
 ### 网络方案

- multus-cni 作为网络选择插件
  * 控制Pod 选flannel 网络
  * 业务Pod 选neutron 网络
  * [EAS-43012](https://easystack.atlassian.net/browse/EAS-43012)

- kuryr-kubernetes
  * cni 连接neutron网络
  * [EAS-43015](https://easystack.atlassian.net/browse/EAS-43015)
  

 ### 存储方案

- 容量型: 类似于虚机数据盘
  * namespace下的resourceQuota 限制容量+统计使用量

- 性能型: 对接高性能低时延
  * namespace下的resourceQuota 限制容量+统计使用量
  * csi插件: https://review.easystack.cn/#/c/45941/

- hostha 处理: [EAS-60400](https://easystack.atlassian.net/browse/EAS-60400)


 ### 安全容器使用GPU

- 当前版本只支持直通，不支持vGPU
 
- k8s-device-plugin 实现GPU上报+调度  https://review.easystack.cn/#/admin/projects/easystack/k8s-device-plugin

- Guest Image预安装 nvidia driver

- nvidia-container-hook 将Guest中的设备、driver挂载到容器里面