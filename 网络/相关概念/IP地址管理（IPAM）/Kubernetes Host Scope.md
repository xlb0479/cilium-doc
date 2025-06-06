# Kubernetes Host Scope

`ipam: kubernetes`启用该模式，将地址分配交给每个节点处理。根据每个节点的`PodCIDR`，由Kubernetes分配IP。

![img](../img/k8s_hostscope.webp)

略