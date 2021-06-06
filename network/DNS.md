### 租户/namespace dns隔离 

- coredns per namespace: https://github.com/kubernetes/dns/issues/132
- 每个ns部署coredns, 配置 kubernetes namespaces only exposes the k8s namespaces listed: https://coredns.io/plugins/kubernetes/
- 每个ns下面的Pods配置 Pod's DNS Config指向所属ns的coredns

### ddns