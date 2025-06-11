# BGP控制面Resource

Cilium的BGP控制面由一组自定义资源管理，提供BGP peer、策略、发布的配置。

包括：

- `CiliumBGPClusterConfig`：定义BGP实例和peer配置。
- `CiliumBGPPeerConfig`：公共的BGP peer配置。用于多个peer。
- `CiliumBGPAdvertisement`：注入到BGP路由表中的前缀。
- `CiliumBGPNodeConfigOverride`：节点相关的BGP配置。

各资源关系如下：

![img](./img/image.png)

## BGP集群配置

`CiliumBGPClusterConfig`定义的BGP配置作用于一个或多个集群节点，具体要看它的`nodeSelector`。每个`CiliumBGPClusterConfig`定义了一个或多个BGP实例，由它们的`name`唯一标识。

一个BGP实例可以有多个peer。每个peer由其`name`唯一标识。Peer自治号以及peer地址由`peerASN`和`peerAddress`定义。Peer的配置由`peerConfigRef`指定，它引用了一个peer配置资源。`peerConfigRef`中的`Group`和`kind`是可选的，默认是`cilium.io`、`CiliumBGPPeerConfig`。

> 警告，`CiliumBGPPeeringPolicy`和`CiliumBGPClusterConfig`不要一起用。如果都加了，Cilium根据node selector选中的相同节点会以`CiliumBGPPeeringPolicy`的优先。

下面的例子中配置了一个名为`instance-65000`的BGP实例，其下有两个peer。

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPClusterConfig
metadata:
  name: cilium-bgp
spec:
  nodeSelector:
    matchLabels:
      rack: rack0
  bgpInstances:
  - name: "instance-65000"
    localASN: 65000
    peers:
    - name: "peer-65000-tor1"
      peerASN: 65000
      peerAddress: fd00:10:0:0::1
      peerConfigRef:
        name: "cilium-peer"
    - name: "peer-65000-tor2"
      peerASN: 65000
      peerAddress: fd00:11:0:0::1
      peerConfigRef:
        name: "cilium-peer"
```

## BGP Peer配置

`CiliumBGPPeerConfig`用来定义一个BGP peer配置。多个peer可以共用同一个配置，引用同一个`CiliumBGPPeerConfig`。

`CiliumBGPPeerConfig`资源包含以下配置：

- MD5密码
- 定时器
- EBGP Multihop
- 优雅重启
- Transport
- 地址族

下面是一个`CiliumBGPPeerConfig`的例子。后面会过一下里面的每个配置。

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeerConfig
metadata:
  name: cilium-peer
spec:
  timers:
    holdTimeSeconds: 9
    keepAliveTimeSeconds: 3
  authSecretRef: bgp-auth-secret
  ebgpMultihop: 4
  gracefulRestart:
    enabled: true
    restartTimeSeconds: 15
  families:
    - afi: ipv4
      safi: unicast
      advertisements:
        matchLabels:
          advertise: "bgp"
```

### MD5密码

`AuthSecretRef`可以设置一个基于[RFC-2385](https://www.rfc-editor.org/rfc/rfc2385.html)的TCP MD5密码。

它的值应该是引用BGP密钥namespace（用Helm安装的话默认是`kube-system`）中的一个Secret的名字。其中应该包含一个名为`password`的key。

BGP密钥保存在指定的namespace中，这样可以把权限范围控制起来。Helm会默认为Cilium做好相关配置。

下面是一个创建Secret的例子：

```shell
kubectl create secret generic -n kube-system --type=string secretname --from-literal=password=my-secret-password
```

如果要改namespace，需要改Helm的chart value`bgpControlPlane.secretNamespace.name`。如果需要自动创建namespace，还要将`bgpControlPlane.secretNamespace.create`设置为`true`。

由于TCP MD5是对数据包的header进行签名，如果Cilium对会话做了地址转换就不能用了（就是说Cilium Agent的Pod IP地址必须是BGP对端能看到的）。

如果密码不对，或者header被改了，TCP就连不上。Cilium Agent的日志中会有`dial: i/o timeout`而非特别具体的错误。

如果Cilium找不到`authSecretRef`，BGP会话就会使用一个空密码，agent会有如下日志：

```text
level=error msg="Failed to fetch secret \"secretname\": not found (will continue with empty password)" component=manager.fetchPeerPassword subsys=bgp-control-plane
```

### 定时器

支持以下定时器参数。详见[RFC4271](https://datatracker.ietf.org/doc/html/rfc4271)。

名称|配置字段|默认值
-|-|-
ConnectRetryTimer|`connectRetryTimeSeconds`|120
HoldTimer|`holdTimeSeconds`|90
KeepaliveTimer|`keepAliveTimeSeconds`|30

如果是在数据中心部署，通常建议把`HoldTimer`和`KeepaliveTimer`设置的更小一点，可以更快的发现问题。比如最小的可能值设置为`holdTimeSeconds=9`、`keepAliveTimeSeconds=3`。

与对端失联后如果要确保快速恢复，可以降低`connectRetryTimeSeconds`（比如设置成`5`或者更小）。因为内部实现的时候会有随机的抖动，`ConnectRetryTimer`的真实值在`[ConnectRetryTimeSeconds, 2 * ConnectRetryTimeSeconds)`这个区间。

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeerConfig
metadata:
  name: cilium-peer
spec:
  timers:
    connectRetryTimeSeconds: 5
    holdTimeSeconds: 9
    keepAliveTimeSeconds: 3
```

### EBGP Multihop

eBGP默认将BGP数据包的IP TTL设置为1。通常不建议改这个东西，但有的时候可能也要改一下。比如BGP对端是位于不同子网的一个路由器，那可能就要改成大于1的。

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeerConfig
metadata:
  name: cilium-peer
spec:
  ebgpMultihop: 4 # <-- specify the TTL value
```