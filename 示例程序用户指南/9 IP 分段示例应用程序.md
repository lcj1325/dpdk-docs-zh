# 9. IP 碎片示例应用程序

IPv4 分段应用程序是使用数据平面开发套件 (DPDK) 进行数据包处理的简单示例。该应用程序通过 IPv4 和 IPv6 数据包分段进行 L3 转发。

## 9.1.概述

该应用程序演示了如何使用零拷贝缓冲区进行数据包分段。

初始化和运行时路径与 [L2 Forwarding Sample Application (in Real and Virtualized Environments)](https://doc.dpdk.org/guides/sample_app_ug/l2_forward_real_virtual.html) 的初始化和运行时路径非常相似。本指南重点介绍了这两个应用程序之间的差异。

与 L2 转发示例应用程序存在三个主要区别：
- 第一个区别是 IP 分段示例应用程序使用间接缓冲区。
- 第二个区别是转发决策是根据从输入数据包的 IP 标头中读取的信息做出的。
- 第三个区别是应用程序通过卸载标志来区分 IP 和非 IP 流量。

最长前缀匹配（IPv4 为 LPM，IPv6 为 LPM6）表用于存储/查找与该 IP 地址关联的传出端口号。任何不匹配的数据包都会转发到始发端口。

默认情况下，支持最大 9.5 KB 的输入帧大小。在转发之前，输入 IP 数据包会被分段以适合“标准”以太网* v2 MTU（1500 字节）。

## 9.2.编译应用程序

该应用程序位于 `ip_fragmentation` 子目录中。

## 9.3.运行应用程序

创建 LPM 对象并加载从全局 l3fwd_ipv4_route_array 和 l3fwd_ipv6_route_array 表中读取的预配置条目。对于每个输入数据包，数据包转发决策（即数据包的输出接口的标识）将作为 LPM 查找的结果。如果 IP 数据包大小大于默认输出 MTU，则输入数据包将被分段，并通过输出接口发送多个分段。

应用程序使用：
```bash
./<build_dir>/examples/dpdk-ip_fragmentation [EAL options] -- -p PORTMASK [-q NQ]
```

其中：
- -p PORTMASK 是要配置的端口的十六进制位掩码
- -q NQ 是每个 lcore 的队列（=端口）数量（默认为 1）

要在 2 个端口 (0,2) 上具有 2 个 lcore (2,4) 的 Linux 环境中运行示例，每个 lcore 有 1 个 RX 队列：

```shell
./<build_dir>/examples/dpdk-ip_fragmentation -l 2,4 -n 3 -- -p 5
EAL: coremask set to 14
EAL: Detected lcore 0 on socket 0
EAL: Detected lcore 1 on socket 1
EAL: Detected lcore 2 on socket 0
EAL: Detected lcore 3 on socket 1
EAL: Detected lcore 4 on socket 0
...

Initializing port 0 on lcore 2... Address:00:1B:21:76:FA:2C, rxq=0 txq=2,0 txq=4,1
done: Link Up - speed 10000 Mbps - full-duplex
Skipping disabled port 1
Initializing port 2 on lcore 4... Address:00:1B:21:5C:FF:54, rxq=0 txq=2,0 txq=4,1
done: Link Up - speed 10000 Mbps - full-duplex
Skipping disabled port 3IP_FRAG: Socket 0: adding route 100.10.0.0/16 (port 0)
IP_FRAG: Socket 0: adding route 100.20.0.0/16 (port 1)
...
IP_FRAG: Socket 0: adding route 0101:0101:0101:0101:0101:0101:0101:0101/48 (port 0)
IP_FRAG: Socket 0: adding route 0201:0101:0101:0101:0101:0101:0101:0101/48 (port 1)
...
IP_FRAG: entering main loop on lcore 4
IP_FRAG: -- lcoreid=4 portid=2
IP_FRAG: entering main loop on lcore 2
IP_FRAG: -- lcoreid=2 portid=0
```

要在 2 个端口 (0,2) 上具有 1 个 lcore (4) 的 Linux 环境中运行示例，每个 lcore 有 2 个 RX 队列：
```
./<build_dir>/examples/dpdk-ip_fragmentation -l 4 -n 3 -- -p 5 -q 2
```

要测试应用程序，应在流生成器中设置与 l3fwd_ipv4_route_array 和/或 l3fwd_ipv6_route_array 表中的值匹配的流。

默认的 l3fwd_ipv4_route_array 表为：
```c
struct l3fwd_ipv4_route l3fwd_ipv4_route_array[] = {
		{RTE_IPV4(100,10,0,0), 16, 0},
		{RTE_IPV4(100,20,0,0), 16, 1},
		{RTE_IPV4(100,30,0,0), 16, 2},
		{RTE_IPV4(100,40,0,0), 16, 3},
		{RTE_IPV4(100,50,0,0), 16, 4},
		{RTE_IPV4(100,60,0,0), 16, 5},
		{RTE_IPV4(100,70,0,0), 16, 6},
		{RTE_IPV4(100,80,0,0), 16, 7},
};
```

默认的 l3fwd_ipv6_route_array 表为：
```c
static struct l3fwd_ipv6_route l3fwd_ipv6_route_array[] = {
	{{1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1}, 48, 0},
	{{2,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1}, 48, 1},
	{{3,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1}, 48, 2},
	{{4,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1}, 48, 3},
	{{5,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1}, 48, 4},
	{{6,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1}, 48, 5},
	{{7,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1}, 48, 6},
	{{8,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1}, 48, 7},
};
```

例如，对于输入的IPv4数据包，目的地址为：100.10.1.1，数据包长度为9198字节，七个 IPv4 数据包将从端口 #0 发送到目标地址 100.10.1.1：其中 6 个数据包的长度为 1500 字节，1 个数据包的长度为 318 字节。IP 分段示例应用程序提供基本的 NUMA 支持，因为所有内存结构都分配在具有活动 lcore 的所有套接字上。

有关运行应用程序和环境抽象层 (EAL) 选项的一般信息，请参阅 DPDK 入门指南。