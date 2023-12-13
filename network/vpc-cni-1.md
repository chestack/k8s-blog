## VPC-CNI 调研

### what is vpc-cni

- 知道vpc是啥, 知道cni是啥，但vpc-cni是个啥?
- vpc-cni 是针对cloud上的k8s集群形态, **将cloud中IaaS层的SDN网络能力通过CNI的形式给k8s集群中的容器使用**. 这里的**能力**包含两部分:
  - SDN丰富的功能
  - underlay网络的性能
- 对于这种CNI的思路, 各大云厂商都有各自的实现. 这里是参考了 [amazon-vpc-cni-k8s](https://github.com/aws/amazon-vpc-cni-k8s) 的命名, 觉得这个名字 信达雅, 所以文章里把这种网络CNI叫做 vpc-cni, 鼓掌.

### why vpc-cni

为什么要这样做呢? 不是有cilium、calico 吗.

- 性能问题
  - cilium、calico... 作为通用CNI插件的设计, 默认跨节点组网方案是走隧道: cilium(vxlan、geneve)、calico(ipip、vxlan); 同时支持配置BGP. 
  其他CNI也一样, 总结下来就是, k8s集群容器跨节点网络访问 只有两种模式: **隧道**、**路由**.

  - 隧道模式 对节点间网络连接要求低, 3层可达即可, 但因为隧道的封包解包 性能会受影响.
    - 如果是部署在裸金属上的k8s集群, 依赖硬件网卡的[offload](./TX.md)可降低影响, 所以一般影响不大. 
    - 但对于部署在cloud上vm里的k8s集群, VM之间网络连接已经是隧道模式, 容器集群的CNI会再加一层隧道, Pod间跨节点通信就会是两层 overlay, **双层牛堡的满足感**, 再加上2500 CPU性能的buff叠加, 封包解包性能损耗 酸爽感人. 一般优化思路是开启vm多队列、提升vm本身的pps能力(其实也是看Infrastructure的SDN性能), 这个就要各显神通了.
  
  - 路由模式 因为没有封包解包性能好些, 但对节点间网络连接有要求.
    - 要么节点在同一子网, 可以在节点上直接建立路由规则 例如flannel gw模式 [要求节点间2层可达][1]; 
    - 要么两个节点间虽不同subnet, 但通过网关路由可达, 这就要求网关能及时且正确更新路由规则. 一般两种思路: BGP 或者 自己实现routes operator, 但无论哪一种都需要在网关(vrouter)上做事情, cloud上vrouter一般不支持BGP 也不会允许vm里随意修改vrouter的路由配置.

- 访问问题
  - 通用CNI环境, k8s集群外无法直接访问Pod IP，只能访问到Pod所在节点再跳转到Pod.
  - vpc-cni 将IaaS层的subnet、ip池等网络资源直接给Pod使用.  同一VPC内k8s集群外的工作负载 可以更方便的访问Pod,  VPC以外可以通过FIP 访问Pod 服务, 可以通过SNAT对出口流量做审计.

- ACL问题
  - 通用CNI的网络ACL 通过 network policy功能, 标准 network policy 只作用在集群内访问控制.
  - 使用vpc-cni, SDN的网络ACL 和 网关防火墙功能可以在集群外对Pod做流量控制.

vpc-cni核心思路是, 将容器网络和容器所在虚拟机的网络做扁平化压缩为一层，同时复用SDN的网络能力   

### 云厂商方案调研

- [阿里Terway设计文档](https://github.com/AliyunContainerService/terway/blob/main/docs/design.md) **首推这个Terway的设计文档, 因为是设计文档不是产品介绍文档, 所以细节介绍的更清晰, 更好理解, 且有代码辅助**
  - terway作者 深入介绍terway(内含ppt下载方式) [[5]], 文档里能看到terway迭代到最新版本主要支持的是以下模式:
  - ENI多IP, 节点内部采用 veth策略路由 模式.
  - ENI多IP, 节点内部采用 ipvlan l2 模式.

- [腾讯云网络概述][2]
  - GlobalRouter, 路由模式.
  - VPC-CNI, VPC 弹性网卡给容器用.
  - Cilium-Overlay, 第三方节点添加到 TKE 集群的网络管理.
  - [tkestack/galaxy](https://github.com/tkestack/galaxy/tree/master)

- [华为云网络模型对比][3]
  - 容器隧道网络, ovs隧道
  - VPC网络, VPC路由
  - 云原生网络2.0, VPC弹性网卡/弹性辅助网卡
  

- [字节 cello](https://github.com/volcengine/cello)


- [amazon-vpc-cni-k8s](https://github.com/aws/amazon-vpc-cni-k8s)
  
  [Learning how AWS implement AWS VPC CNI](https://www.slideshare.net/hongweiqiu/learning-how-aws-implement-aws-vpc-cni)


- Lyft, [cni-ipvlan-vpc-k8s](https://github.com/lyft/cni-ipvlan-vpc-k8s)
  
  [Announcing cni-ipvlan-vpc-k8s: IPvlan overlay-free Kubernetes Networking in AWS](https://eng.lyft.com/announcing-cni-ipvlan-vpc-k8s-ipvlan-overlay-free-kubernetes-networking-in-aws-95191201476e)


- [网易Kubernetes网络方案][4]


- 白牌 blog
  - http://www.iceyao.com.cn/post/2020-05-12-k8s_network_with_openstack_2/
  - https://juejin.cn/post/6844903801057443853
  

[1]: https://stackoverflow.com/questions/45293321/why-host-gw-of-flannel-requires-direct-layer2-connectivity-between-hosts
[2]: https://github.com/tencentyun/qcloud-documents/blob/master/product/%E8%AE%A1%E7%AE%97%E4%B8%8E%E7%BD%91%E7%BB%9C/%E5%AE%B9%E5%99%A8%E6%9C%8D%E5%8A%A1/%E6%8E%A7%E5%88%B6%E5%8F%B0%E6%8C%87%E5%8D%97%EF%BC%88%E6%96%B0%E7%89%88%EF%BC%89/%E7%BD%91%E7%BB%9C%E7%AE%A1%E7%90%86/%E5%AE%B9%E5%99%A8%E7%BD%91%E7%BB%9C%E6%A6%82%E8%BF%B0.md
[3]: https://github.com/huaweicloudDocs/cce/blob/master/cn.zh-cn/%E7%94%A8%E6%88%B7%E6%8C%87%E5%8D%97/%E5%AE%B9%E5%99%A8%E7%BD%91%E7%BB%9C%E6%A8%A1%E5%9E%8B%E5%AF%B9%E6%AF%94.md
[4]: https://sq.sf.163.com/blog/article/223878660638527488?hmsr=toutiao.io&utm_campaign=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io
[5]: https://developer.aliyun.com/article/755848?spm=a2c6h.12873639.article-detail.10.5d3424adfdhEI8
