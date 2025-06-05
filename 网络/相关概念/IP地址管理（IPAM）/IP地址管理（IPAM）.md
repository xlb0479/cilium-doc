# IP地址管理（IPAM）

IP地址管理就是负责分配和管理网络中的IP地址（容器和其他资源）。为满足不同需求给出了多种模式：

功能|Kubernetes Host Scope|Cluster Scope（默认）|Multi-Pool
-|-|-|-
隧道路由|√|√|×
直接路由|√|√|√
CIDR配置|Kubernetes|Cilium|Cilium
单集群多个CIDR|×|√|√
单节点多CIDR|×|×|√
动态CIDR/IP分配|×|×|√

对于运行中的集群不要改IPAM模式，否则可能会导致运行中的负载持久性的失去网络。最安全的方式就是在刚开始部署的时候去调整IPAM模式。如果你对如何扩展Cilium来支持IPAM模式迁移感兴趣，参见[GitHub issue 27164](https://github.com/cilium/cilium/issues/27164)。