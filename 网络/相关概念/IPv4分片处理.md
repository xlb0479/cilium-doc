# IPv4 Fragment处理

默认情况下Cilium通过eBPF来实现IP分片跟踪，让那些不支持分段的协议（比如UDP）可以透明地在网络上传输比较大的消息。该功能支持以下配置：

- `--enable-ipv4-fragment-tracking`：启停IPv4分片跟踪。默认开启。
- `--bpf-fragments-map-max`：使用IP分片的最大并发连接数。默认值见[eBPF Map](../eBPF%20Datapath/eBPF%20Map.md)。

> 该功能还处于beta。