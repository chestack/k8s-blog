## CNI 工作原理

k8s网络包括三个部分：Pod网络、svc网络、network policy<br>
CNI负责解决Pod网络问题，Pod网络也是其k8s里其他网络的基础。

#### 调用流程: kubelet-->CRI-->CNI-->network plugin

![cni-call](pics/cni-process.png) 

整体流程如上，调用过程相对简单，简单在Pod网络的setup过程是单机行为，不涉及分布式和网络的调用。<br>
调用流程中涉及到的主要代码模块由蓝色框标出，按照先后顺序:

- kubelet.go, 配置cni入口目录，调用CRI的RunPodSandbox()方法发起Pods创建
- cri/../sandbox_run.go, kubelet只是发起创建sandbox对network无感知，CRI真正发起创建网络 setupPodNetwork()
- cni/libcni/api.go, CRI调用CNI的入口，CRI只负责创建network namespace，后续工作交给CNI
- cni/pkg/skel/skel.go, network plugin入口，真正开始Add/Del network的地方

总体流程: kubelet-->CRI-->CNI-->network plugin

### CNI Deep Dive

##### cni/SPEC.md[[1]]

- Add/Del Parameters: Container ID，ns path，extra args
- Result: Interfaces list，IP configuration assigned to each interface， DNS information
- 对应上面两个spec的定义，下面是具体方法和数据结构的定义
    ```$xslt
    // AddNetworkList executes a sequence of plugins with the ADD command
    func (c *CNIConfig) AddNetworkList(ctx context.Context, list *NetworkConfigList, rt *RuntimeConf) (types.Result, error) {
        var err error
        var result types.Result
        for _, net := range list.Plugins {
            result, err = c.addNetwork(ctx, list.Name, list.CNIVersion, net, result, rt)
            if err != nil {
                return nil, err
            }
        }
     
    // NetConf describes a network.
    type NetConf struct {
        CNIVersion string `json:"cniVersion,omitempty"`
    
        Name         string          `json:"name,omitempty"`
        Type         string          `json:"type,omitempty"`
        Capabilities map[string]bool `json:"capabilities,omitempty"`
        IPAM         IPAM            `json:"ipam,omitempty"`
        DNS          DNS             `json:"dns"`
    
        RawPrevResult map[string]interface{} `json:"prevResult,omitempty"`
        PrevResult    Result                 `json:"-"`
    } 
    
    type RuntimeConf struct {
        ContainerID string
        NetNS       string
        IfName      string
        Args        [][2]string
        CapabilityArgs map[string]interface{}
    
        // DEPRECATED. Will be removed in a future release.
        CacheDir string
    }
    ```
- Network Configuration Lists
    ```$xslt
    {
      "name":"cni0",
      "cniVersion":"0.3.1",
      "plugins":[
        {
          "type":"flannel",
          "delegate":{
            "forceAddress":true,
            "hairpinMode": true,
            "isDefaultGateway":true
          }
        },
        {
          "type":"portmap",
          "capabilities":{
            "portMappings":true
          }
        }
      ]
    }
    ```
- plugins是数组结构，cni会按数组顺序依次处理每一个plugin
- 每个plugin可以自定义自己识别的args，上面例子里flannel plugin包含了*delegate*参数，对应代码
    ```$xslt
    type NetConf struct {
        types.NetConf
    
        SubnetFile    string                 `json:"subnetFile"`
        DataDir       string                 `json:"dataDir"`
        Delegate      map[string]interface{} `json:"delegate"`
        RuntimeConfig map[string]interface{} `json:"runtimeConfig,omitempty"`
    }
    ```
   
##### cni is more than interface

通常理解的interface只会定义: 方法名、参数、返回值(对比CRI grpc API)。cni里包含了部分必须的实现, 串联整个调用流程
- libcni/api.go, AddNetworkList(), DelNetworkList()
- pkg/skel/skel.go,  实现了network plugin Add/Del/Check 三个基本功能的框架


##### cni plugins[[2]]
- main plugins(interface-creating): bridge, macvlan, ipvlan
- ipam plugins: host-local, dhcp
- meta plugins: flannel(flannel plugin的实现在这里和coreos/flanneld配合工作), portmap


### network plugin example 代码分析[[3]]

#### flannel

- 入口 plugins/meta/flannel/flannel.go
```$xslt
func main() {
	skel.PluginMain(cmdAdd, cmdCheck, cmdDel, version.All, bv.BuildString("flannel"))
}
```

- Add(bridge, host-local) plugins/meta/flannel/flannel_linux.go
```$xslt
func doCmdAdd(args *skel.CmdArgs, n *NetConf, fenv *subnetEnv) error {
	n.Delegate["name"] = n.Name

	if !hasKey(n.Delegate, "type") {
		n.Delegate["type"] = "bridge"
	}
    ...
	n.Delegate["ipam"] = map[string]interface{}{
		"type":   "host-local",
		"subnet": fenv.sn.String(),
		"routes": []types.Route{
			{
				Dst: *fenv.nw,
			},
		},
	}

	return delegateAdd(args.ContainerID, n.DataDir, n.Delegate)
}
```

- Delegate to bridge.cmdAdd() plugins/main/bridge/bridge.go
```$xslt
func cmdAdd(args *skel.CmdArgs) error {
    ...
}
```
- setupBridge(n), 配置网桥
- setupVeth(netns, br, args.IfName, n.MTU, n.HairpinMode, n.Vlan)， 配置veth pairs
- ipam.ExecAdd(n.IPAM.Type, args.StdinData)，获取ip
- calcGateways(result, n)，计算生成网关信息
- ipam.ConfigureIface(args.IfName, result)，配置ip地址，ifname，route rules
- arping.GratuitousArpOverIface(ipc.Address.IP, *contVeth)，网卡启动ARP
- err = ensureAddr(br, gws.family, &gw, n.ForceAddress), 为网桥配置ip地址 设置为网关
- ip.SetupIPMasq(&ipc.Address, chain, comment)，使用iptables配置容器访问外网的masquerade规则

#### portmap

当Pod定义里使用了hostPort[[4]]时，需要配置portMapping功能，不推荐使用

```$xslt
  containers:
  - args:
    - --api
    - --kubernetes
    - --logLevel=INFO
    image: hub-cn-shanghai-2.kce.ksyun.com/ksyun/traefik:latest
    imagePullPolicy: Always
    name: traefik-ingress-lb
    ports:
    - containerPort: 80
      hostPort: 80
      name: http
      protocol: TCP
    - containerPort: 8080
      hostPort: 8080
      name: admin
      protocol: TCP
```

[1]: https://github.com/containernetworking/cni/blob/master/SPEC.md
[2]: https://github.com/containernetworking/plugins
[3]: https://www.flftuu.com/2019/03/01/CNI%E7%BD%91%E7%BB%9C%E6%9E%B6%E6%9E%84%E8%A7%A3%E6%9E%90/
[4]: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#containerport-v1-core