# Cilium的BGP控制面

BGP控制面可以让Cilium将路由通过[BGP](https://datatracker.ietf.org/doc/html/rfc4271)协议发布给路由器。在支持BGP的环境中，可以让Pod、Service的网络能够从集群外部访问。

> 想了解Cilium BGP的更深层的知识可以看看[这个视频](https://www.youtube.com/watch?v=Tv0R6VxyWhc)。

## 安装

### Helm

将Helm的`bgpControlPlane.enabled`设置为true。

```shell
$ helm upgrade cilium cilium/cilium --version 1.17.4 \
    --namespace kube-system \
    --reuse-values \
    --set bgpControlPlane.enabled=true
$ kubectl -n kube-system rollout restart ds/cilium
```

支持IPv4/IPv6的单双栈。BGP控制面只能发布Cilium配置的地址族。比如Cilium Agent只配置了IPv6地址族，那就不能发布IPv4的路由。反之亦然。

## 配置BGP控制面

一共两种方式。一是用遗留的`CiliumBGPPeeringPolicy`，二是用新的`CiliumBGPClusterConfig`。目前这两种都支持，以后会废弃前者。