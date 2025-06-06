# Cluster Scope（默认）

这种模式按节点分配PodCIDR，每个节点用host-scope分配器来分配IP。因此有点类似于[Kubernetes Host Scope](./Kubernetes%20Host%20Scope.md)。不同之处在于，后者是用Kubernetes的`v1.Node`来分配每个节点的PodCIDR，而Cilium operator是用`v2.CiliumNode`来管理每个节点的PodCIDR。好处就是不依赖Kubernetes了。

## 架构

![img](../img/cluster_pool.png)

如果Kubernetes无法实现分配PodCIDR，或者需要更多控制的时候，这种模式就比较好。

这种模式下，Cilium agent启动时会等待所有`v2.CiliumNode`中指定的`podCIDRs`生效。

字段|描述
-|-
`spec.ipam.podCIDRs`|IPv4或IPv6的PodCIDR

## 配置

实操见[基于CRD的Cilium Cluster-Pool IPAM](../../Kubernetes网络/配置IPAM模式/基于CRD的Cilium%20Cluster-Pool%20IPAM.md)

### 扩充cluster pool

`clusterPoolIPv4PodCIDRList`的列表不要改，否则可能导致无法预料的行为。如果池子耗尽了，往里新加元素就好。掩码最小到`/30`，一般推荐最小是`/29`。至于为什么说要增不要改，是为了让分配器为每个CIDR块保留2个IP用作网络和广播地址。`clusterPoolIPv4MaskSize`也不要改。

## 排错

### 分配错误

看`status.ipam.operator-status`的`Error`字段：

```shell
kubectl get ciliumnodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.ipam.operator-status}{"\n"}{end}'
```

### 检查CIDR冲突

Pod CIDR默认是`10.0.0.0/8`。如果跟你节点所在网络冲突了，就无法跟其他节点连通了。外出流量都会被认作是要去节点上的某个Pod，而不是去其他节点。

可以用以下方法解决：

- 明确设置`clusterPoolIPv4PodCIDRList`加以区分
- 修改节点的CIDR