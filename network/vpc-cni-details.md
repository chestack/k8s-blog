## VPC-CNI Deep Dive
 
### Multi-homing 问题

参照 [vpc-cni 网络拓扑](./vpc-cni-architecture.md), 虚机会挂载多张网卡, 其中一张作为主网卡(k8s node 中记录的ip地址来源), 及多张数据网卡(承载容器网络进出虚机), 节点上通过策略路由配置容器流量从数据网卡出虚机. 

虚机多张网卡的情况下可能会遇到 流量从A网卡进 B网卡出的问题。某些blog里会把这类问题 称为 network multi-homing问题(根据wiki "multi-homing"定义可能不准确，但是可以作为key word 搜索)。

具体到容器的访问场景

- 东西向，因为软SDN的ARP代答功能, pod <--> pod 会从数据网卡进 数据网卡出。
- 南北向(nodeport), 流量会从主网卡进入, 如上所说策略路由 默认会从数据网卡出(这里nodeport具体会分为 endpoints在本节点 和 不在本节点两种情况, 出问题的场景是前者 local endpoint，后面默认讨论这个场景).
  
  虚机网卡开启有状态安全组的情况下, TCP三次握手建链过程因为数据网卡没有第一次握手的信息，对出的流量会drop.

#### mark

对于上面提到的容器的nodeport场景，解决思路也很清晰: 主网卡进保持主网卡出；数据网卡进数据网卡出。 

思路清晰 落地难, 具体怎么做呢？
- 通常的方案是对不同流量数据包打mark区分是哪个网卡进 + 策略路由判断mark决定哪个网卡出。参照[Packet Mark In a Cloud Native World](https://arthurchiao.art/blog/packet-mark-in-a-cloud-native-world-zh/)
  
  但是, packet mark 跨network namespace是不生效的。在容器的场景下，如果nodeport包到节点就打mark，进入pod之后再出来 之前的mark信息就不在了。[linux-packet-mark-across-network-namespaces](https://unix.stackexchange.com/questions/704511/linux-packet-mark-across-network-namespaces)

- 封回路转. nodeport默认会做SNAT 即SRC IP是节点IP，即会换成主网卡IP地址. 参照[k8s 文档](https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-type-nodeport) .
基于这个前提, 可以只在host network ns里对pod出流量做判断加mark，即如果是包是从容器网卡出 且目标地址是主网卡地址, 则认为这个是nodeport的回包.
  
- 基于以上分析和尝试，最终方案如下
```cgo
iptables -t mangle -A PREROUTING -i lxc+ -d <本节点ens3地址> -j MARK --set-mark 137
ip rule add from all fwmark 137 lookup main priority 110<local 之后 111之前>
```

#### rp_filter
上面解决的是 nodeport 进入Pod之后再出来的流量做区分。在nodeport PREROUTING<-->Routing Decision<-->FORWARD， Routing Decision这里会遇到rp_filter的问题。[rp_filter的介绍](https://blog.csdn.net/u014644574/article/details/130118306)

- nodeport场景: client(ip1) --> ens3(ip2) --> iptables 做DNAT --> lxc容器(ip3)

- ens3.rp_filter 作用在routing decision(prerouting之后  forward之前)， 是对 ens3收到的包  做回包(转发) 时候做 filter 看是否要drop，drop的判断条件是 如果回包(转发) 不走ens3
- DNAT到 容器ip之后, vpc-cni 的策略路由， 匹配的路由规则是  from <容器IP> lookup 11, 即从 数据网卡出，所以drop。注: 上面说的 fwmark 策略路由优先级更高, 但rp_filter这里是静态判断。
- rp_filter=0, 不校验


总结, 解决vpc-cni nodeport(local endpoint)问题的三板斧: rp_filter、fwmark、policy routing.


### 预热

### 调度

### 固定IP