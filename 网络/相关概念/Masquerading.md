# Masquerading

Pod用的IPv4地址通常都是从RFC1918私有地址块分配的，外面都看不到。Cilium会自动将离开集群的流量的源地址伪装成集群节点的IP，这样到了外面就认识了。

![img](./img/masquerade.png)

可以用`enable-ipv4-masquerade: false`、`enable-ipv6-masquerade: false`关闭伪装。

## 配置

### 设置可路由的CIDR

默认是排除掉落入节点本地CIDR的目的IP。如果Pod IP能在更大的网络中进行路由，可以通过`ipv4-native-routing-cidr: 10.0.0.0/8`、`ipv6-native-routing-cidr: fd00::/100`设置，在该范围内的IP**不会**被伪装。

如果是公有云环境，不设置`ipv4-native-routing-cidr`的话Cilium会自动检测VPC的CIDR作为本地路由。Cilium不会伪装本地可路由网络中的源地址，因为两端无需NAT就能通信。因此如果启用伪装，从Pod去往其他同一个VPC中的非集群资源的流量就不会做源地址伪装，会直接路由。

### 设置伪装接口

看下面

## 实现模式

### 基于eBPF

> 注意
> IPv6的BPF伪装还处于beta阶段。遇到问题到GitHub发issue。IPv4的BPF伪装已经可以用于生产环境了。

这种模式的实现效率最高。可以通过helm的`bpf.masquerade=true`启用。

