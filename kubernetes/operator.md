## Operator Deep Dive

- client-go/informer vs. k8s.io/code-generator vs. sample-controller vs. controller-runtime vs. kubebuilder

- list-watch
  * python-client, requests 包, timeout配置[[3]]
```$xslt
WARNING kuryr_kubernetes.k8s_client [-] 60s without data received from watching /api/v1/pods. Retrying the connection with resourceVersion=3420351.
```
  * go-client, k8s>=1.5(streaming http/2)[[5]]
  * server端如何处理client的watch [[6]]
```$xslt
// cacherWatch implements watch.Interface
type cacheWatcher struct {
	sync.Mutex
	input chan watchCacheEvent
	result chan watch.Event
	filter FilterFunc
	stopped bool
	forget func(bool)
}
```

  * watch的是resource changes, 是k8s组件消息机制, 组件间交互用的
```$xslt
curl -k -H "Content-Type: application/json" http://127.0.0.1:8080/api/v1/namespaces/75de16c57d5e48da9467de5f9dedcc16/pods?watch=1

curl -k -H "Content-Type: application/json" http://127.0.0.1:8080/api/v1/namespaces/75de16c57d5e48da9467de5f9dedcc16/pods?watch=1?resourceVersion=3420351
```

  * events 是 k8s 内部组件工作产生的messages，当做log给人看的. events kv的方式存在etcd里. k8s GC清理events
```bash
curl -k -H "Content-Type: application/json" http://127.0.0.1:8080/api/v1/namespaces/75de16c57d5e48da9467de5f9dedcc16/events?watch=1

ETCDCTL_API=3 etcdctl --endpoints=https://<ip>:2379 get / --prefix --keys-only

--event-ttl duration Default: 1h0m0s
```

- watch到一个resource变化之后，单独一个gorotuine 处理吗？处理失败怎么办？
  * MaxConcurrentReconciles, is defined are implementations from the controller-runtime [[7]] [[8]]
  * re-add to the queue failed reconcile keys [[9]]
  
- Add Or Update [[12]]

- finalizer, 定义CRD的删除逻辑, [[10]] [[11]]
  * force delete Namespaces stuck in Terminating [[13]]

- python client without informer [[4]]

- kube-apiserver listwatch etcd
```bash
etcdctl watch /registry/pods/75de16c57d5e48da9467de5f9dedcc16/aaaab-55c9df9c56-gnz6d

kubectl get pods -n 75de16c57d5e48da9467de5f9dedcc16
```

- operator HA
  * 是否存在AA模式？
  * leader election sidecar [[14]] 问题: kuryr-controller crash, leader-election work, 不会切换
  * controller-runtime 支持leader election [[15]] , 推荐!

[3]: https://bugs.launchpad.net/kuryr-kubernetes/+bug/1842689
[4]: https://github.com/kubernetes-client/python/issues/868
[5]: https://juejin.cn/post/6844903593519251464
[6]: https://developer.aliyun.com/article/680204
[7]: https://github.com/kubernetes-sigs/controller-runtime/issues/616
[8]: https://github.com/operator-framework/operator-sdk/issues/1938
[9]: https://github.com/kubernetes-sigs/kubebuilder/pull/228
[10]: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#finalizers
[11]: https://segmentfault.com/a/1190000020359577
[12]: https://github.com/kubernetes-sigs/kubebuilder/issues/37
[13]: https://stackoverflow.com/questions/55853312/how-to-force-delete-a-kubernetes-namespace
[14]: https://kubernetes.io/blog/2016/01/simple-leader-election-with-kubernetes/#leader-election-with-sidecars
[15]: https://github.com/kubernetes-sigs/controller-runtime/pull/118