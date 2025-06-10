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
- 