## VPC-CNI

### what is vpc-cni

- 知道vpc是啥, 知道cni是啥，但vpc-cni是个啥? 没错, 又造一个新名词, 因神秘而好奇, 这种行为也叫产品化.
- 好好说话, vpc-cni 是针对cloud上的k8s集群形态, **将cloud中Infrastructure的SDN网络能力作为CNI给k8s集群中的容器使用**, 各个公有云厂商都支持方案, 因为公有云才有cloud 才有vpc.
- 当然, 各个公有云厂商的名字不一样, 这里是参考了 [amazon-vpc-cni-k8s](https://github.com/aws/amazon-vpc-cni-k8s) 的命名, 觉得这个名字 信达雅, 所以文章里把这种网络CNI叫做 vpc-cni, 鼓掌.

### why vpc-cni

为什么要这样做呢? 不是有cilium、calico吗.

- cilium、calico... 作为通用CNI插件的设计, 默认跨节点组网方案是走隧道: cilium(vxlan、geneve)、calico(ipip、vxlan); 同时支持配置BGP. 
  其他CNI也一样, 总结下来就是, k8s集群容器跨节点网络访问 只有两种模式: **隧道**、**路由**.

- 隧道模式 对节点间网络连接要求低, 3层可达即可, 但因为隧道的封包解包 性能会受影响.
  - 如果是部署在裸金属上的k8s集群, 依赖硬件网卡的[offload](./TX.md)可降低影响, 所以一般影响不大.
  - 但对于部署在cloud上vm里的k8s集群, VM之间网络连接已经是隧道模式, 容器集群的CNI会再加一层隧道, Pod间跨节点通信就会是两层 overlay, **双层牛堡的满足感**. 一般优化思路是开启vm多队列、提升vm本身的pps能力(其实也是看Infrastructure的SDN性能), 这个就要各显神通了.

- 路由模式 因为没有封包解包性能好些, 但对节点间网络连接有要求.
  - 要么节点在同一子网, 可以在节点上直接建立路由规则 例如flannel gw模式 [要求节点间2层可达][1]; 
  - 要么两个节点间虽不同subnet, 但通过网关路由可达, 这就要求网关能及时且正确更新路由规则. 一般两种思路: BGP 或者 自己实现routes operator, 但无论哪一种都需要在网关(vrouter)上做事情, cloud上vrouter一般不支持BGP 也不会允许vm里随意修改vrouter的路由配置.
  
- 除了跨节点访问, 在ACL上, vm有一层security group, pod之间有network policy，两层ACL的实现, "层层加码".

综上, **对于部署在vm上(部署在cloud上vpc里)的k8s集群**, 通用CNI为了屏蔽底层网络差异, 提出通用的方案但牺牲了网络性能. 哪里有问题哪里就有机会, 所以有了vpc-cni, 简单逻辑就是 **基于底层SDN能力订制的CNI > 通用CNI**.

### 公有云厂商方案调研

- [阿里Terway设计文档](https://github.com/AliyunContainerService/terway/blob/main/docs/design.md) **首推这个Terway的设计文档, 因为是设计文档不是产品介绍文档, 所以细节介绍的更清晰, 更好理解, 且有代码辅助**
  - VPC, Pod网段不同于节点的网络的网段，通过Aliyun VPC路由表打通不同节点间的容器网络。
  - ENI, 容器的网卡是Aliyun弹性网卡，Pod的网段和宿主机的网段是一致的。
  - ENI多IP, 一个Aliyun弹性网卡可以配置多个辅助VPC的IP地址，将这些辅助IP地址映射和分配到Pod中，这种Pod的网段和宿主机网段也是一致的。

- [腾讯云网络概述][2]
  - GlobalRouter, 路由模式.
  - VPC-CNI, VPC 弹性网卡给容器用.
  - Cilium-Overlay, 第三方节点添加到 TKE 集群的网络管理.

- [华为云网络模型对比][3]
  - 容器隧道网络, ovs隧道
  - VPC网络, VPC路由
  - 云原生网络2.0, VPC弹性网卡/弹性辅助网卡

- [amazon-vpc-cni-k8s](https://github.com/aws/amazon-vpc-cni-k8s)
  
  [Learning how AWS implement AWS VPC CNI](https://www.slideshare.net/hongweiqiu/learning-how-aws-implement-aws-vpc-cni)


- [网易Kubernetes网络方案][4]

### ES vpc-cni 思路/方案

根据以上分析 和 竞品调研, 跨节点网络通信基本两个思路: 
- vrouter 网关上配置路由
- 两层压扁到一层, 底层SDN为Pod提供网卡, 要么VM里多网卡, 要么vm单网卡多IP

除了基础的跨节点通信, 还需要考虑k8s网络API能力的实现(k8s API能力要基于跨节点通信的实现来实现), 包括以下几个方面

- pod <--> svc访问, 因为svc是基于host的iptables, 所以如果Pod使用底层SDN网卡, 网络流量默认会绕过vm host network stack
  
- network policy, 前面提到 pod和vm ACL的层层加码, 所以这里两个思路
  
  - SDN要即支持 security group, 也支持 network policy, 对应到产品里ovn作为SDN, 对于两个实现分别可以参照 neutron 和 kube-ovn
  - network policy的支持交给其他插件, 如calico的felix 或者 cilium
  
- QoS, 这个能力没有标准的 k8s网络API(不同于 network policy、gateway API、ingress API), 所以支持起来不需要两个语义入口, 直接按照SDN的QoS能力实现即可, 大概思路
  
  - ovn实现
  - tc规则实现, 参照 bandwidth CNI


[1]: https://stackoverflow.com/questions/45293321/why-host-gw-of-flannel-requires-direct-layer2-connectivity-between-hosts
[2]: https://github.com/tencentyun/qcloud-documents/blob/master/product/%E8%AE%A1%E7%AE%97%E4%B8%8E%E7%BD%91%E7%BB%9C/%E5%AE%B9%E5%99%A8%E6%9C%8D%E5%8A%A1/%E6%8E%A7%E5%88%B6%E5%8F%B0%E6%8C%87%E5%8D%97%EF%BC%88%E6%96%B0%E7%89%88%EF%BC%89/%E7%BD%91%E7%BB%9C%E7%AE%A1%E7%90%86/%E5%AE%B9%E5%99%A8%E7%BD%91%E7%BB%9C%E6%A6%82%E8%BF%B0.md
[3]: https://github.com/huaweicloudDocs/cce/blob/master/cn.zh-cn/%E7%94%A8%E6%88%B7%E6%8C%87%E5%8D%97/%E5%AE%B9%E5%99%A8%E7%BD%91%E7%BB%9C%E6%A8%A1%E5%9E%8B%E5%AF%B9%E6%AF%94.md
[4]: https://sq.sf.163.com/blog/article/223878660638527488?hmsr=toutiao.io&utm_campaign=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io
