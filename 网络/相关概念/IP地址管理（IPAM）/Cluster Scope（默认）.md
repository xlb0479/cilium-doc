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