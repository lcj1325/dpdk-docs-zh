# 9. 跟踪库

## 9.1.概述

跟踪是一种用于了解正在运行的软件系统中发生的情况的技术。用于跟踪的软件称为跟踪器，其概念类似于磁带录音机。录制时，放置在软件源代码中的特定检测点会生成保存在巨大磁带上的事件：跟踪文件。稍后可以在跟踪查看器中打开跟踪文件，以使用时间戳和多核视图来可视化和分析跟踪事件。这种机制将有助于解决广泛的问题，例如多核同步问题，延迟测量，找出 CPU 空闲时间等分析后信息，否则这些信息很难获得。

跟踪通常与日志记录相比较。然而，跟踪器和记录器是两种不同的工具，有两个不同的用途。跟踪器旨在记录比日志消息更频繁发生的低级别事件，通常在每秒数千个范围内，而执行开销非常小。日志记录更适合对不太频繁的事件进行非常高级的分析：用户访问、异常情况（例如错误和警告）、数据库事务、即时消息通信等。简而言之，日志记录是跟踪可以满足的众多用例之一。

## 9.2. DPDK跟踪库功能

- 一个在控制和快速路径 API 中添加跟踪点的框架，对性能的影响最小。典型的跟踪开销约为 20 个周期，仪器开销为 1 个周期。
- 在运行时启用和禁用跟踪点。
- 在任何时间点将跟踪缓冲区保存到文件系统。
- 支持 `overwrite` 和 `discard` 跟踪模式操作。
- 基于字符串的跟踪点对象查找。
- 基于正则表达式和/或通配符启用和禁用一组跟踪点。
- 以 `Common Trace Format (CTF)` 生成跟踪。 CTF 是一种开源跟踪格式，与 `LTTng` 兼容。有关详细信息，请参阅[通用跟踪格式](https://diamon.org/ctf/)。

## 9.3.如何添加跟踪点？

本部分将引导您完成添加简单跟踪点的详细信息。

### 9.3.1.创建跟踪点头文件

```c
#include <rte_trace_point.h>

RTE_TRACE_POINT(
       app_trace_string,
       RTE_TRACE_POINT_ARGS(const char *str),
       rte_trace_point_emit_string(str);
)
```

上面的宏创建了 app_trace_string 跟踪点。用户可以为跟踪点选择任何名称。但是，在 DPDK 库中添加跟踪点时，必须遵循 `rte_<library_name>_trace_[<domain>_]<name>` 命名约定。示例为 `rte_eal_trace_generic_str`、`rte_mempool_trace_create`。

`RTE_TRACE_POINT` 宏从上面的定义扩展为以下函数模板：

```c
static __rte_always_inline void
app_trace_string(const char *str)
{
        /* Trace subsystem hooks */
        ...
        rte_trace_point_emit_string(str);
}
```

此跟踪点的使用者可以调用 `app_trace_string(const char *str)` 将跟踪事件发送到跟踪缓冲区。

### 9.3.2.注册跟踪点

```c
#include <rte_trace_point_register.h>

#include <my_tracepoint.h>

RTE_TRACE_POINT_REGISTER(app_trace_string, app.trace.string)
```

上面的代码片段将 `app_trace_string` 跟踪点注册到跟踪库。这里，`my_tracepoint.h`是用户在第一步[创建追踪点头文件](https://doc.dpdk.org/guides/prog_guide/trace_lib.html#create-tracepoint-header-file)中创建的头文件。

RTE_TRACE_POINT_REGISTER 的第二个参数是跟踪点的名称。该字符串将用于跟踪点查找或正则表达式和/或基于全局的跟踪点操作。跟踪点函数及其名称不要求相似。但是，建议使用相似的名称以获得更好的命名约定。

> note:
> 在包含 rte_trace_point.h 标头之前，必须包含 rte_trace_point_register.h 标头。

> note:
> `RTE_TRACE_POINT_REGISTER` 定义 `rte_trace_point_t` 跟踪点对象的占位符。对于通用跟踪点或公共头文件中使用的跟踪点，用户必须在库 `.map` 文件中导出 `__<trace_function_name>` 符号，以便在共享构建中在库外使用此跟踪点。例如，__app_trace_string 将是上例中的导出符号。

## 9.4.快速路径跟踪点

为了避免快速路径代码中的性能影响，该库引入了 `RTE_TRACE_POINT_FP`。在快速路径代码中添加跟踪点时，用户必须使用 `RTE_TRACE_POINT_FP` 而不是 `RTE_TRACE_POINT`。

`RTE_TRACE_POINT_FP` 默认情况下编译出来，可以使用介子构建的`enable_trace_fp` 选项启用它。

## 9.5.事件记录模式

事件记录模式是跟踪缓冲区的一个属性。跟踪库公开了以下模式：

- 覆盖 
    当跟踪缓冲区已满时，新的跟踪事件将覆盖跟踪缓冲区中现有的捕获事件。
- 丢弃 
    当跟踪缓冲区已满时，新的跟踪事件将被丢弃。

该模式可以在应用程序启动时使用 EAL 命令行参数 `--trace-mode` 进行配置，也可以使用 `rte_trace_mode_set()` API 在运行时进行配置。

## 9.6.跟踪文件位置

在 `rte_trace_save()` 或 `rte_eal_cleanup()` 调用时，库将跟踪缓冲区保存到文件系统。默认情况下，跟踪文件存储在 `$HOME/dpdk-traces/rte-yyyy-mm-dd-[AP]M-hh-mm-ss/` 中。它可以通过 `--trace-dir=<目录路径>` EAL 命令行选项覆盖。

有关详细信息，请参阅跟踪 EAL 命令行选项的 [EAL 参数](https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html)。

## 9.7.查看并分析记录的事件

一旦跟踪目录可用，用户就可以查看/检查记录的事件。

您可以使用许多工具来读取 DPDK 跟踪：

1. `babeltrace` 是一个转换跟踪格式的命令行实用程序；它支持 DPDK 跟踪库生成的格式、CTF 以及可进行 grep 编辑的基本文本输出。 babeltrace 命令是开源 Babeltrace 项目的一部分。
2. `Trace Compass` 是一个图形用户界面，用于查看和分析任何类型的日志或跟踪，包括 DPDK 跟踪。

### 9.7.1.使用 babeltrace 命令行工具

列出跟踪的所有记录事件的最简单方法是将其路径传递给 `babeltrace`，不带任何选项：

```bash
babeltrace </path-to-trace-events/rte-yyyy-mm-dd-[AP]M-hh-mm-ss/>
```

`babeltrace` 在给定路径中递归地查找所有跟踪并打印所有事件，并按时间顺序合并它们。

您可以将 `babeltrace` 的输出通过管道传输到 grep(1) 等工具中以进行进一步过滤。下面的示例仅 `grep ethdev` 的事件：

```bash
babeltrace /tmp/my-dpdk-trace | grep ethdev
```

您可以将 `babeltrace` 的输出通过管道传输到 wc(1) 等工具中以对记录的事件进行计数。下面的示例计算 `ethdev` 事件的数量：

```bahs
babeltrace /tmp/my-dpdk-trace | grep ethdev | wc --lines
```

### 9.7.2.使用tracecompass GUI工具

`Tracecompass` 是另一个用于查看/分析 DPDK 跟踪的工具，它提供事件的图形视图。与 `babeltrace` 一样，`tracecompass` 也提供了一个搜索特定事件的接口。要使用`tracecompass`，至少需要执行以下步骤：

- 将tracecompass 安装到本地主机。有适用于 Linux、Windows 和 OS-X 的变体。
- 启动tracecompass，这将打开一个带有跟踪管理界面的图形窗口。
- 使用 `File->Open Trace` 选项打开跟踪，然后选择要查看/分析的元数据文件。

有关更多详细信息，请参阅[跟踪罗盘](https://www.eclipse.org/tracecompass/)。

## 9.8.快速开始

本节将引导您详细了解生成跟踪和查看跟踪的详细信息。

- 启动 dpdk 测试：
```bash
echo "quit" | ./build/app/test/dpdk-test --no-huge --trace=.*
```
- 使用 babeltrace 查看器查看痕迹：
```bash
babeltrace $HOME/dpdk-traces/rte-yyyy-mm-dd-[AP]M-hh-mm-ss/
```

## 9.9.实施细节

由于 DPDK 跟踪库旨在生成使用 `Common Trace Format (CTF)` 的跟踪。 CTF 规范由以下单元组成来创建跟踪。

- `Stream` 数据包的顺序
- `Packet` 标题和一个或多个事件。
- `Event` 标头和有效负载。

有关详细信息，请参阅[通用跟踪格式](https://diamon.org/ctf/)。

实施细节大致分为以下几个方面：

### 9.9.1.跟踪元数据创建

### 9.9.2.痕迹记忆

### 9.9.3.跟踪内存布局

Table 9.1 Trace memory layout.
packet.header
packet.context
trace 0 header
trace 0 payload
trace 1 header
trace 1 payload
trace N header
trace N payload

#### 9.9.3.1. packet.header

Table 9.2 Packet header layout.
uint32_t magic
rte_uuid_t uuid

#### 9.9.3.2. packet.context

Table 9.3 Packet context layout.
uint32_t thread_id
char thread_name[32]

#### 9.9.3.3. trace.header

Table 9.4 Trace header layout.
event_id [63:48]
timestamp [47:0]


跟踪标头为 64 位，由 48 位时间戳和 16 位事件 ID 组成。

`packet.header` 和 `packet.context` 将在创建跟踪内存时写入慢速路径中。当调用tracepoint函数时，将发出`trace.header`和trace有效负载。

## 9.10.局限性

- `rte_trace_point_emit_blob()` 函数可以捕获长度为 `RTE_TRACE_BLOB_LEN_MAX` 字节的最大 blob。如果应用程序需要捕获超过 `RTE_TRACE_BLOB_LEN_MAX` 字节，则可以多次调用 `rte_trace_point_emit_blob()`，且长度小于或等于 `RTE_TRACE_BLOB_LEN_MAX`。
- 如果传递给 `rte_trace_point_emit_blob()` 的长度小于 `RTE_TRACE_BLOB_LEN_MAX`，则跟踪中的尾随 `(RTE_TRACE_BLOB_LEN_MAX - len)` 字节将设置为零。

