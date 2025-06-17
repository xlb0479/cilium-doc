# Iptable的使用

根据Linux内核版本的不同，eBPF datapath会实现不同的功能。如果某种需要的功能不可用，对应的实现方式就会使用遗留的iptable来实现。

## kube-proxy互操作

下面展示了kube-proxy设置的iptable规则和Cilium设置的iptable规则是如何集成的。

![img4](./img/image4.png)