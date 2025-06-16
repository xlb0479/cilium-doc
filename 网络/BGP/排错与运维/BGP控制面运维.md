# BGP控制面运维

一些关于BGP控制面操作的指导。

## BGP Cilium CLI

### 安装

安装最新版本的Cilium CLI。Cilium CLI是可以用来装Cilium的，也可以用来检查Cilium的安装情况，开启或关闭各种特性（比如clustermesh、Hubble）。

```shell
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

用`cilium bgp`查看BGP状态。

```shell
# cilium bgp --help
Access to BGP control plane

Usage:
  cilium bgp [command]

Available Commands:
  peers       Lists BGP peering state
  routes      Lists BGP routes

Flags:
  -h, --help   help for bgp

Global Flags:
      --context string             Kubernetes configuration context
      --helm-release-name string   Helm release name (default "cilium")
      --kubeconfig string          Path to the kubeconfig file
  -n, --namespace string           Namespace Cilium is running in (default "kube-system")

Use "cilium bgp [command] --help" for more information about a command.
```

### Peers

`cilium bgp peers`可以查看kubernetes集群中所有节点当前的对等连接状态。

```shell
# cilium bgp peers
Node                                     Local AS   Peer AS   Peer Address   Session State   Uptime   Family         Received   Advertised
bgpv2-cplane-dev-service-control-plane   65001      65000     fd00:10::1     established     33m26s   ipv4/unicast   2          2
                                                                                                      ipv6/unicast   2          2
bgpv2-cplane-dev-service-worker          65001      65000     fd00:10::1     established     33m25s   ipv4/unicast   2          2
                                                                                                      ipv6/unicast   2          2
```

也可以用来验证BGP会话状态，看看是不是`established`，以及广播到对端的路由数量。

### Routes

`cilium bgp routes`可以展示详细的本地BGP路由表，以及对每个对等广播的路由情况。

```shell
# cilium bgp routes available ipv4 unicast
Node                                     VRouter   Prefix        NextHop   Age      Attrs
bgpv2-cplane-dev-service-control-plane   65001     10.1.0.0/24   0.0.0.0   46m45s   [{Origin: i} {Nexthop: 0.0.0.0}]
bgpv2-cplane-dev-service-worker          65001     10.1.1.0/24   0.0.0.0   46m45s   [{Origin: i} {Nexthop: 0.0.0.0}]
```

也可以直接查看每个对等的广播情况。

```shell
# cilium bgp routes advertised ipv4 unicast
Node                                     VRouter   Peer         Prefix        NextHop          Age     Attrs
bgpv2-cplane-dev-service-control-plane   65001     fd00:10::1   10.1.0.0/24   fd00:10:0:1::2   47m0s   [{Origin: i} {AsPath: 65001} {Communities: 65000:99} {MpReach(ipv4-unicast): {Nexthop: fd00:10:0:1::2, NLRIs: [10.1.0.0/24]}}]
bgpv2-cplane-dev-service-worker          65001     fd00:10::1   10.1.1.0/24   fd00:10:0:2::2   47m0s   [{Origin: i} {AsPath: 65001} {Communities: 65000:99} {MpReach(ipv4-unicast): {Nexthop: fd00:10:0:2::2, NLRIs: [10.1.1.0/24]}}]
```

根据你配置的[CiliumBGPAdvertisement](../Cilium的BGP控制面/配置BGP控制面/控制面Resource.md#bgp广播advertisements)进行对比验证。

### Policies

Cilium BGP安装了GoBGP策略，管理每个对等广播和BGP属性。但因为这属于内部的实现细节，所以没有通过CLI暴露出来。但是出于调试的目的，可以用cilium-dbg CLI来查看安装的BGP策略。

```shell
/home/cilium# cilium-dbg bgp route-policies
VRouter   Policy Name          Type     Match Peers      Match Prefixes (Min..Max Len)   RIB Action   Path Actions
65001     65000-ipv4-PodCIDR   export   fd00:10::1/128   10.1.0.0/24 (24..24)            accept       AddCommunities: [65000:99]
65001     65000-ipv6-PodCIDR   export   fd00:10::1/128   fd00:10:1::/64 (64..64)         accept       AddCommunities: [65000:99]
65001     allow-local          import                                                    accept
```

## CiliumBGPClusterConfig的状态

这东西可能会从`.status.conditions`报告一些配置上的错误。目前有如下condition。

**Condition**|**描述**
-|-
`cilium.io/NoMatchingNode`|`.spec.nodeSelector`没有选中任何节点
`cilium.io/MissingPeerConfigs`|`spec.bgpInstances[].peers[].peerConfigRef`引用的PeerConfig不存在。
`cilium.io/ConflictingClusterConfig`|存在另一个CiliumBGPClusterConfig选中了相同的节点。

## CiliumBGPPeerConfig的状态

同上，也是会报告一些错误，目前有以下。

**Condition**|**描述**
-|-
`cilium.io/MissingAuthSecret`|`.spec.authSecretRef`引用的Secret找不到。

## CiliumBGPNodeConfig的状态

启用BGP控制面后，根据`CiliumBGPClusterConfig`中的节点选择器选中的每一个Cilium节点，都会得到对应的`CiliumBGPNodeConfig`资源。`CiliumBGPNodeConfig`资源就是对应节点的BGP配置源，由Cilium operator管理。

`CiliumBGPNodeConfig`的状态反映了BGP的实时运行状态。可以通过这一点实现自动化或相关监控。

在下面的例子中，可以看到`bgpv2-cplane-dev-service-worker`这个节点的BGP实例状态。

```shell
# kubectl describe ciliumbgpnodeconfigs bgpv2-cplane-dev-service-worker
Name:         bgpv2-cplane-dev-service-worker
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  cilium.io/v2alpha1
Kind:         CiliumBGPNodeConfig
Metadata:
  Creation Timestamp:  2024-10-17T13:59:44Z
  Generation:          1
  Owner References:
    API Version:     cilium.io/v2alpha1
    Kind:            CiliumBGPClusterConfig
    Name:            cilium-bgp
    UID:             f0c23da8-e5ca-40d7-8c94-91699cf1e03a
  Resource Version:  1385
  UID:               fc88be94-37e9-498a-b9f7-a52684090d80
Spec:
  Bgp Instances:
    Local ASN:  65001
    Name:       65001
    Peers:
      Name:          65000
      Peer ASN:      65000
      Peer Address:  fd00:10::1
      Peer Config Ref:
        Group:  cilium.io
        Kind:   CiliumBGPPeerConfig
        Name:   cilium-peer
Status:
  Bgp Instances:
    Local ASN:  65001
    Name:       65001
    Peers:
      Established Time:  2024-10-17T13:59:50Z
      Name:              65000
      Peer ASN:          65000
      Peer Address:      fd00:10::1
      Peering State:     established
      Route Count:
        Advertised:  2
        Afi:         ipv4
        Received:    2
        Safi:        unicast
        Advertised:  2
        Afi:         ipv6
        Received:    2
        Safi:        unicast
      Timers:
        Applied Hold Time Seconds:  90
        Applied Keepalive Seconds:  30
Events:                             <none>
```

## 关闭CRD状态报告

CRD的状态报告对排错很有帮助，通常都选择开启它们。但是当集群规模很大，或者BGP策略很多的时候，CRD状态报告可能会对API server带来比较大的压力。可以将Helm的`bgpControlPlane.statusReport.enabled`设置成`false`来关闭它。此时就会关闭状态报告，并清空已经报告出来的状态。

## 日志

BGP控制面日志可以在Cilium operator（仅限BGPv2）和Cilium agent日志中找到。

在operator中日志会打上`subsys=bgp-cp-operator`标记。可以用它来过滤出相关的日志：

```shell
kubectl -n kube-system logs <cilium operator pod name> | grep "subsys=bgp-cp-operator"
```

在agent中日志则是打上了`subsys=bgp-control-plane`标记：

```shell
kubectl -n kube-system logs <cilium agent pod name> | grep "subsys=bgp-control-plane"
```

## Metrics

略

## 重启Agent

重启Cilium agent会导致BGP会话断开，因为BGP speaker集成在Cilium agent中。Agent起来之后会立刻恢复BGP会话。但是，当Cilium agent停止时，广播出去的路由会从BGP对端中删掉。所以会临时出现Service和Pod连不上的问题。可以启用[优雅重启](../Cilium的BGP控制面/配置BGP控制面/控制面Resource.md#优雅重启)，在agent重启过程中维持Pod和Service的流量转发。

## Cilium的升级和降级

对Cilium进行升级或降级时，必须重启所有agent。重启部分上面已经说过了。

如果启用了BGP控制面，在升级之前，要提前把agent镜像拉下来，做好[准备工作]()。拉镜像很慢，而且由于网络问题也比较容易出错。如果拉镜像的时间过长，可能会超出优雅重启时间（`restartTimeSeconds`），导致BGP对端删除路由。

## 节点停机

当你需要对某个节点进行停机维护时，参照下面的步骤，避免数据包的丢失。

1. 抽干（drain）当前节点，剔除所有负载。这会从Service中删除所有该节点上的Pod，确保那些`externalTrafficPolicy=Cluster`的Service不会把流量转过来。

    ```shell
    kubectl drain <node-name> --ignore-daemonsets
    ```

2. 重新配置BGP会话，修改或删除该节点在CiliumBGPPeeringPolicy或CiliumBGPClusterConfig中的对应标签。此时会关闭该节点上所有的BGP会话。

    ```shell
    # Assuming you select the node by the label enable-bgp=true
    kubectl label node <node-name> --overwrite enable-bgp=false
    ```

3. 等待BGP对端删除指向该节点的路由。在此期间BGP对端可能仍会发送流量到该节点。如果不等BGP对端删除路由就停机，就会干扰到`externalTrafficPolicy=Cluster`的Service的流量。

4. 节点停机。

在第3步中，你可能无法检查对端状态，但又想知道具体要等多久。此时可以按照下面的方式大致估算出等待时间：

- 如果你没有开启BGP优雅重启，BGP对端会在第2步之后立刻删除路由。
- 如果启用了BGP优雅重启，可能会有两种情况。
    - 如果BGP对端支持带通知的优雅重启（[RFC 8538](https://datatracker.ietf.org/doc/html/rfc8538.html)），它会等Stale Timer（[RFC 8538#section-4.1](https://datatracker.ietf.org/doc/html/rfc8538.html#section-4.1)）到期后删除路由。
    - 如果BGP对端不支持带通知的优雅重启，会在第2步后立刻删除路由，这是因为当你去掉了节点的标签后，BGP控制面会立刻向对端发送BGP通知。

上面的方法估计出的是一个理论值，真实时间肯定是依赖于BGP对端的具体实现。理想情况下是提前跟负责网络的同学一块儿查看一下对端路由器的真实情况。

> 警告，即便按照上面的步骤来，一些原定发往该节点的Service流量依然可能会被重置（TCP Reset），因为路由删除并进行ECMP rehashing之后，流量会被重定向到另一个节点，而新的节点可能会选择一个不同的endpoint。

## 异常情况

这里列出一些常见的异常情况，并给出对应的解决办法。

### Cilium Agent挂了

如果agent挂了，BGP会话就会断开，因为BGP speaker是集成在agent内部的。Cilium agent重启后会立即恢复BGP会话。但是在这期间，广播出去的路由会被对端删掉。结果就是临时性的无法访问Pod和Service。

#### 解决办法

处理该问题的推荐方法就是启用[优雅重启](../Cilium的BGP控制面/配置BGP控制面/控制面Resource.md#优雅重启)。这样可以让BGP对端保留一段时间路由。由于datapath在此期间仍保持active，这样就不会导致Pod或Service无法访问了。

如果你无法使用BGP优雅重启，也可以采取如下步骤，要看你具体使用了什么样的路由：

##### PodCIDR路由

如果你广播了PodCIDR路由，那么异常节点上的Pod是无法被外界访问的。如果异常只出现在集群中的部分节点上，可以drain掉这些节点，使Pod迁移到其他节点。

##### Service路由

如果你广播了Service路由，负载均衡（KubeProxy或者是Cilium KubeProxyReplacement）可能就无法从外部访问了。而且上游路由器的ECMP重新散列，导致正在进行中的连接可能会被重定向到不同的节点。当负载均衡遇到了未知的流量，它就要选择一个新的endpoint。根据负载均衡的算法不同，流量可能会被转向到一个跟之前不同的endpoint上，继而导致连接被reset。

如果上游的路由器支持[Resilient Hashing](https://www.juniper.net/documentation/us/en/software/junos/interfaces-ethernet-switches/topics/topic-map/resillient-hashing-lag-ecmp.html)的ECMP，启用了的话也许会让进行中的连接继续被转发到同一个节点上。启用Cilium的[Maglev Consistent Hashing](../../Kubernetes网络/替换kube-proxy.md#maglev一致性哈希)也会有同样的效果，因为它可以提升所有节点为相同流量选择相同endpoint的几率。但只对`externalTrafficPolicy: Cluster`有效。如果是`local`，那就无法避免异常节点上的连接被转发到另一个节点的情况了，这些连接都会被reset。

### 节点挂了

如果节点挂了，它上面的BGP会话也就都断了。对端回立即删除它广播的路由，如果开了优雅重启那就等待相应的时间。但如果你广播了`externalTrafficPolicy=cluster`的Service的路由，那么后者就有问题了，因为它会在重启等待时间（默认120秒）内继续将流量转发到这个有问题的节点。

#### 解决办法

##### 非主动停机

如果节点是非主动停机，那就没有什么太好的办法。可以选择不用BGP的优雅重启，这就是在异常发现速度和Cilium Pod重启时的稳定性之间进行权衡了。

关闭优雅重启可以让BGP对端更快的撤销路由。即便节点停机时没有BGP Notification，或者也没有TCP连接关闭，撤销路由的最长时间也就是BGP的hold time。如果开了优雅重启，BGP对端可能就要等hold time + restart time这段时间才会撤销路由。

##### 主动停机

参照[节点停机](#节点停机)做就好。

### 对等连接断了

如果对等连接断了，通常来说BGP会话和datapath的连通性也就断了。但在一段时间内可能会出现datapath连通性没有了，而BGP会话还在，并且路由依然被广播出来了。这就会让BGP对端将流量发送到异常的链路上，导致丢包。这个时间段的长度取决于链路以及BGP的配置。

如果直连节点链路断开，BGP会话一般就会立即断开了，因为Linux内核会探测到链路异常，立即断开TCP会话。如果非直连节点链路断开，BGP会话会等待hold timer到时再断开，默认90秒。

#### 解决办法

如果要更快的检测到链路异常，可以缩短`holdTimeSeconds`和`keepAliveTimeSeconds`。最小值分别是`3`和`1`。通常用来加快异常检测的方法是BFD（Bidirectional Forwarding Detection），但Cilium目前还不支持。

### Cilium Operator挂了

这东西要负责将`CiliumBGPClusterConfig`翻译成每个节点的`CiliumBGPNodeConfig`。如果这玩意挂了，BGP控制面的生成就会有问题。

同样，PodCIDR、LoadBalancer IP的分配也会出问题，前者依赖IPAM，后者依赖LB-IPAM。因此PodCIDR和Service虚拟IP的分配和撤销都会出问题。

#### 解决办法

从BGP层面来说没啥解决办法。可以对Operator做高可用部署，这样会好一点。

### Service的后端全挂了

如果由于掉电或者配置错误导致Service的所有后端全都失联，此时BGP控制面的行为依赖于Service的`externalTrafficPolicy`。如果是`Cluster`，根据`CiliumBGPPeeringPolicy`或`CiliumBGPClusterConfig`的配置，Service的虚拟IP会继续向外广播。如果`Local`，那么直接停掉广播，因为只有存在活着的后端，对应节点的虚拟IP才会向外广播。

#### 解决办法

BGP没啥办法。这是Kubernetes的事儿，可以试试PodDisruptionBudget。