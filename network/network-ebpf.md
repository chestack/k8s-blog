### ebpf 在linux中有哪些可以load代码的 hook点

- xdp
- tc
- cgroup ？？ 
- socket

Understanding the eBPF networking features in RHEL 9
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_networking/assembly_understanding-the-ebpf-features-in-rhel-9_configuring-and-managing-networking


[深入理解 Cilium 的 eBPF 收发包路径](https://arthurchiao.art/blog/understanding-ebpf-datapath-in-cilium-zh/)

这里面结合网卡的RX过程, 详细介绍了cilium 可以做ebpf hook的各层.

https://zhuanlan.zhihu.com/p/484859413
xdp --> netif_receive_skb() --> ip_rcv --> tc
tc 的处理发生在 ip_rcv() 之后。也就是说，当数据包经过网络层的 ip_rcv() 函数进行处理之后，如果存在与该数据包相关的 tc 规则，则会按照这些规则对数据包进行分类、过滤、限制、重定向等操作。在这个过程中，通过 Qdisc 机制可以对数据包进行各种复杂的控制操作，从而实现对网络流量的高效、精细化控制。


### 基于ebpf 做 replace kube-proxy 怎么搞

Improving Kubernetes Service Network Performance with Socket eBPF
https://www.alibabacloud.com/blog/improving-kubernetes-service-network-performance-with-socket-ebpf_599446

cilium ebpf(host-services) replace kube-proxy
https://www.tkng.io/services/clusterip/dataplane/ebpf/