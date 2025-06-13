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

BGP对等无法建立的原因有很多，比如BGP能力不匹配或者Peer IP填错了。BGP层面的错误一般都会出现在日志里，但也有一些底层错误可能不会出现在日志里，比如到Peer IP的连通性问题，或者某个eBGP的距离大于1跳，此时用`WireShark`或者`tcpdump`更有效果。

## 添加了新的CiliumBGPPeeringPolicy之后当前BGP会话立刻断连

由于`nodeSelector`的配置问题，同一个节点可能会被多个`CiliumBGPPeeringPolicy`选中。如果应用了多个策略，那么BGP控制面会删除该节点已有的所有状态。此时，首先将最后应用的`CiliumBGPPeeringPolicy`回滚，并检查BGP断连时的节点日志。如果出现了多个策略，会有这样的日志：

```log
level=error msg="Policy selection failed" component=Controller.Reconcile error="more then one CiliumBGPPeeringPolicy applies to this node, please ensure only a single Policy matches this node's labels" subsys=bgp-control-plane
```

此时就要检查`nodeSelector`配置，确保每个节点只关联一个`CiliumBGPPeeringPolicy`。

## 新加的CiliumBGPClusterConfig不管用

跟`CiliumBGPPeeringPolicy`类似，多个`CiliumBGPClusterConfig`根据`nodeSelector`的配置也可能选中相同的节点。如果出现这种情况，Cilium operator会拒绝额外的`CiliumBGPClusterConfig`再去添加`CiliumBGPNodeConfig`资源。`CiliumBGPClusterConfig`的状态会被设置成下面这样：

```yaml
status:
  conditions:
  - lastTransitionTime: "2024-10-26T06:18:04Z"
    message: 'Selecting the same node(s) with ClusterConfig(s): [cilium-bgp-0]'
    observedGeneration: 1
    reason: ConflictingClusterConfigs
    status: "True"
    type: cilium.io/ConflictingClusterConfig
```

## CiliumBGPPeerConfig不起作用

如果`CiliumBGPPeerConfig`不起作用，可能是因为`peerConfigRef`配错了，引用无效。如果找不到引用的`CiliumBGPPeerConfig`，就会出现下面的状态：

```yaml
status:
  conditions:
  - lastTransitionTime: "2024-10-26T06:15:44Z"
    message: 'Referenced CiliumBGPPeerConfig(s) are missing: [peer-cofnig0]'
    observedGeneration: 1
    reason: MissingPeerConfigs
    status: "True"
    type: cilium.io/MissingPeerConfigs
```