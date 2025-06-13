# BGP控制面排错

这里给出一些典型问题及其解决方案。

## 应用了BGP资源，但是无法建立BGP对等

### CiliumBGPPeeringPolicy

看看`CiliumBGPPeeringPolicy`的`nodeSelector`是否选中了正确的节点。最简单的方法就是用`cilium bgp peers`：

```shell
$ cilium bgp peers
Node                              Local AS   Peer AS   Peer Address   Session State   Uptime   Family         Received   Advertised
node0                             65001      65000     10.0.1.1       active          0s       ipv4/unicast   0          0
                                                                                               ipv6/unicast   0          0
```

如果Node匹配正确，即便会话没有建立，也是会显示Node名称和BGP状态的。如果啥也没显示，那么`nodeSelector`可能就有问题。如果Node匹配正确，但是从状态上看没有建立起来，需要检查Cilium和目标对端的状态。

### CiliumBGPClusterConfig

检查每个节点是否有对应的`CiliumBGPNodeConfig`资源。如果没有，看一下Cilium operator的日志，看看有没有什么错误。

和`CiliumBGPPeeringPolicy`类似，如果BGP状态不是`established`，检查`nodeSelector`配置和对端配置。

还有一种可能就是`CiliumBGPClusterConfig`中的`nodeSelector`没有匹配任何节点，可能是因为配错了标签或选择器。此时的状态会出现：

```yaml
status:
  conditions:
  - lastTransitionTime: "2024-10-26T06:15:44Z"
    message: No node matches spec.nodeSelector
    observedGeneration: 2
    reason: NoMatchingNode
    status: "True"
    type: cilium.io/NoMatchingNode
```

## CiliumBGPPeeringPolicy或CiliumBGPClusterConfig选好了节点，但是BGP对等无法建立

可以在Cilium或对等路由器的日志中找到问题根源。BGP控制面的日志中有一个`subsys=bgp-control-plane`这样的字段，可以用来过滤和BGP控制面相关的错误日志：

```shell
$ kubectl -n <your namespace> <cilium pod running on the target node> logs | grep bgp-control-plane
...
level=warning msg="sent notification" Data="as number mismatch expected 65003, received 65000" Key=10.0.1.1 Topic=Peer asn=65001 component=gobgp.BgpServerInstance subsys=bgp-control-plane
```

在上面的例子中，可以看到，BGP会话无法建立时因为配置的`peerASN`和真实的对端ASN不一样导致的。