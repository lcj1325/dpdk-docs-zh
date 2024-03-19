
# 1. DPDK示例应用程序简介

DPDK 示例应用程序是小型独立应用程序，演示了 DPDK 的各种功能。它们可以被视为 DPDK 功能的示例。有兴趣开始使用 DPDK 的用户可以使用该应用程序，尝试其功能，然后对其进行扩展以满足自己的需求。

## 1.1.运行示例应用程序

一些示例应用程序可能在各自的指南中描述了自己的命令行参数，但是所有这些应用程序也共享相同的 EAL 参数。请参阅 [EAL parameters (Linux)](https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html) 或 [EAL parameters (FreeBSD)](https://doc.dpdk.org/guides/freebsd_gsg/freebsd_eal_parameters.html) 以获取可用 EAL 命令行选项的列表。

## 1.2. DPDK 示例应用程序

DPDK 的示例目录中有许多示例应用程序。这些示例的范围从简单到相当复杂，但大多数都是为了演示 DPDK 的一项特定功能而设计的。下面重点介绍了一些重要的示例。
- [Hello World](https://doc.dpdk.org/guides/sample_app_ug/hello_world.html)：与大多数编程框架的介绍一样，Hello World 应用程序是一个很好的起点。Hello World 示例设置 DPDK 环境抽象层 (EAL)，并向每个启用 DPDK 的内核打印一条简单的“Hello World”消息。该应用程序不执行任何数据包转发，但它是测试 DPDK 环境是否已正确编译和设置的好方法。
- [Basic Forwarding/Skeleton Application](https://doc.dpdk.org/guides/sample_app_ug/skeleton.html)：基本转发/骨架包含使用 DPDK 启用基本数据包转发所需的最少量代码。这允许您测试您的网络接口是否与 DPDK 配合使用。
- [Network Layer 2 forwarding](https://doc.dpdk.org/guides/sample_app_ug/l2_forward_real_virtual.html)：网络第 2 层转发或 `l2fwd` 应用程序像简单交换机一样基于以太网 MAC 地址进行转发。
- [Network Layer 2 forwarding](https://doc.dpdk.org/guides/sample_app_ug/l2_forward_event.html): 网络第 2 层转发或 `l2fwd-event` 应用程序像简单交换机一样基于以太网 MAC 地址进行转发。它演示了在单个应用程序下轮询和事件模式IO机制的使用。
- [Network Layer 3 forwarding](https://doc.dpdk.org/guides/sample_app_ug/l3_forward.html)：网络第 3 层转发或 `l3fwd` 应用程序像简单的路由器一样基于 Internet 协议、IPv4 或 IPv6 进行转发。
- [Network Layer 3 forwarding Graph](https://doc.dpdk.org/guides/sample_app_ug/l3_forward_graph.html)：网络第 3 层转发图或 `l3fwd_graph` 应用程序基于 IPv4 进行转发，就像具有 DPDK Graph 框架的简单路由器一样。
- [Hardware packet copying](https://doc.dpdk.org/guides/sample_app_ug/dma.html)：硬件数据包复制或 `dmafwd` 应用程序演示了如何使用 DMAdev 库在两个线程之间复制数据包。
- [Packet Distributor](https://doc.dpdk.org/guides/sample_app_ug/dist_app.html)：数据包分发器演示如何将到达 Rx 端口的数据包分发到不同的内核进行处理和传输。
- [Multi-Process Application](https://doc.dpdk.org/guides/sample_app_ug/multi_process.html)：多进程应用程序展示了两个 DPDK 进程如何使用队列和内存池来共享信息。
- [RX/TX callbacks Application](https://doc.dpdk.org/guides/sample_app_ug/rxtx_callbacks.html)：RX/TX 回调示例应用程序是一个数据包转发应用程序，演示了如何在接收和传输的数据包上使用用户定义的回调。应用程序通过向 RX 和 TX 数据包处理函数添加回调来计算 RX（数据包到达）和 TX（数据包传输）之间的数据包延迟。
- [IPsec Security Gateway](https://doc.dpdk.org/guides/sample_app_ug/ipsec_secgw.html)：IPsec 安全网关应用程序是更接近现实世界示例的最小示例。这也是使用 DPDK Cryptodev 框架的应用程序的一个很好的示例。
- [Precision Time Protocol (PTP) client](https://doc.dpdk.org/guides/sample_app_ug/ptpclient.html): PTP 客户端是现实应用程序的另一个最小实现。在本例中，应用程序是 PTP 客户端，它与 PTP 主时钟通信，以使用 IEEE1588 协议同步网络接口卡 (NIC) 上的时间。
- [Quality of Service (QoS) Scheduler](https://doc.dpdk.org/guides/sample_app_ug/qos_scheduler.html)：QoS 调度程序应用程序演示了如何使用 DPDK 提供 QoS 调度。

接下来的章节中还有更多示例。每个记录的示例应用程序都展示了如何编译、配置和运行应用程序，并解释了代码的主要功能。
