## merbridge

内容参照[merbridge 官方blog](https://merbridge.io/zh/blog/2022/03/01/merbridge-introduce/)

### 数据面iptables流量劫持问题

istio+envoy的架构下, 数据面默认是通过iptables实现流量的劫持, 对单个Pod的流量劫持如下图, [Istio 服务网格的现状与未来](https://jimmysong.io/blog/beyond-istio-oss/)

![istio-route-iptables](../pics/istio-route-iptables.svg)

假设现在有一个服务 A 想要调用非本地主机上的另一个 pod 中的服务 B，可以看到整个调用流程如下图, 会经过四次 iptables, 会损失不少性能.

![process-iptables](../pics/iptables-process.svg)

ebpf如何解决这个问题呢，参照之前的文章[网络RX过程](../network/RX.md), 我们知道xdp/ebpf可以在L2网卡驱动之后将packet交给socket(绕过内核 TCP/IP 协议栈).
上图中通信过程涉及到四个socket: service_A、envoy_A inbound、envoy_A outbound、service_B、envoy_B inbound、envoy_B outboun, 所以我们可以通过ebpf编程实现实现 packets在socket间直达.

- 非同节点服务A 访问 服务B, 经过两次iptables
  ![ebpf-diff-nodes](../pics/ebpf-diff-node.svg)
  
- 同节点服务A 访问 服务B, 可以不经过iptables，socket之间直达
  ![ebpf-same-node](../pics/ebpf-same-node.svg)


### merbridge 实现原理

- 结合[原文](https://merbridge.io/zh/blog/2022/03/01/merbridge-introduce/) 的配图，看代码吧
- 文章里的性能测试结果, 貌似没啥吸引力呢

### merbridge roadmap

- [merbridge CNI 模式](https://merbridge.io/zh/blog/2022/05/18/cni-mode/), 这看起来是cilium mesh的思路, mesh结合cni才能看更多事情, 毕竟1+1>2
- [merbridge + ambient](https://merbridge.io/zh/blog/2022/11/11/ambient-mesh-support/), sidecarless大势所趋, 这条路感觉没毛病. 感觉 istio ambient + merbridge 和 cilium service mesh 殊途同归了, 都是 sidecarless+ebpf, 都是更快、更简单的朴素追求.