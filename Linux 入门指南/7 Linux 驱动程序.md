# 7.Linux驱动程序

不同的包出路模块（PMDs, Packet Processing Modules）可能需要不同的内核驱动程序才能正常工作。根据所使用的 PMDs，应加载相应的内核驱动程序，并将网络端口绑定到该驱动程序。

## 7.1.与内核模块绑定和解除绑定网络端口

```
note: 
使用分流驱动的 PMDs 不应与其内核驱动程序解除绑定。
本节适用于使用 UIO 或 VFIO 驱动程序的 PMDs。
有关更多详细信息，请参阅 [Bifurcated Driver](https://doc.dpdk.org/guides/linux_gsg/linux_drivers.html#bifurcated-driver) 部分。
```

```
note: 
建议在所有情况下都使用 `vfio-pci` 作为 DPDK 绑定端口的内核模块。
如果IOMMU不可用，则可以在 [no-iommu](https://doc.dpdk.org/guides/linux_gsg/linux_drivers.html#vfio-noiommu) 模式下使用`vfio-pci`。
如果由于某种原因 vfio 不可用，则可以使用基于 UIO 的模块 `igb_uio` 和 `uio_pci_generic`。
详细信息请参见 [UIO](https://doc.dpdk.org/guides/linux_gsg/linux_drivers.html#uio) 部分。
```

大多数设备要求 DPDK 使用的硬件与其使用的内核驱动程序解除绑定，并且在应用程序运行之前绑定到 `vfio-pci` 内核模块。对于此类 PMDs，Linux* 控制下的任何网络端口或其他硬件都将被忽略，并且不能被应用程序使用。

要将端口绑定到 vfio-pci 模块以供 DPDK 使用，或者将端口返回给 Linux 控制，`usertools` 子目录中提供了一个名为 `dpdk-devbind.py` 的实用程序脚本。该实用程序可用于提供系统上网络端口当前状态的视图，并从不同的内核模块（包括 VFIO 和 UIO 模块）绑定和取消绑定这些端口。以下是如何使用该脚本的一些示例。可以通过使用 `--help` 或 `--usage` 选项调用脚本来获取脚本及其参数的完整描述。请注意，要使用的 UIO 或 VFIO 内核模块应在运行 `dpdk-devbind.py` 脚本之前加载到内核中。

```
note: 
由于 VFIO 的工作方式，可与 VFIO 一起使用的设备存在一定的限制。
主要取决于 IOMMU 组的工作方式。
任何虚拟功能设备通常都可以单独与 VFIO 一起使用，
但物理设备可能需要将所有端口绑定到 VFIO，或者其中一些端口绑定到 VFIO，
而其他端口则根本不绑定任何东西。

如果您的设备位于 PCI 至 PCI 桥接器后面，则该桥接器将成为您的设备所在的 IOMMU 组的一部分。
因此，桥接驱动程序也应该与桥接 PCI 设备解除绑定，以便 VFIO 能够与桥接器后面的设备一起工作。
```

```
虽然任何用户都可以运行 `dpdk-devbind.py` 脚本来查看网络端口的状态，
但绑定或解除绑定网络端口需要 root 权限。
```

查看系统上所有网络端口的状态：

```
./usertools/dpdk-devbind.py --status

Network devices using DPDK-compatible driver
============================================
0000:82:00.0 '82599EB 10-GbE NIC' drv=vfio-pci unused=ixgbe
0000:82:00.1 '82599EB 10-GbE NIC' drv=vfio-pci unused=ixgbe

Network devices using kernel driver
===================================
0000:04:00.0 'I350 1-GbE NIC' if=em0  drv=igb unused=vfio-pci *Active*
0000:04:00.1 'I350 1-GbE NIC' if=eth1 drv=igb unused=vfio-pci
0000:04:00.2 'I350 1-GbE NIC' if=eth2 drv=igb unused=vfio-pci
0000:04:00.3 'I350 1-GbE NIC' if=eth3 drv=igb unused=vfio-pci

Other network devices
=====================
<none>
```

要将设备 `eth1`, `04:00.1` 绑定到 `vfio-pci` 驱动程序：

```
./usertools/dpdk-devbind.py --bind=vfio-pci 04:00.1
```

或者：

```
./usertools/dpdk-devbind.py --bind=vfio-pci eth1
```

指定设备 ID 时，可以在地址的最后部分使用通配符。要将设备 82:00.0 和 82:00.1 恢复为其原始内核绑定：

```
./usertools/dpdk-devbind.py --bind=ixgbe 82:00.*
```

## 7.2. VFIO

VFIO 是一个依赖 IOMMU 保护的强大且安全的驱动程序。要使用 VFIO，必须加载 `vfio-pci` 模块：

```
sudo modprobe vfio-pci
```

VFIO 内核通常默认存在于所有发行版中，但是请查阅您的发行版文档以确保情况确实如此。

要利用完整的 VFIO 功能，内核和 BIOS 都必须支持并配置为使用 IO 虚拟化（例如 Intel® VT-d）。

```
note: 在大多数情况下，指定“iommu=on”作为内核参数应该足以将 Linux 内核配置为使用 IOMMU。
```

为了在以非特权用户身份运行 DPDK 应用程序时正确操作 VFIO，还应该设置正确的权限。有关更多信息，请参阅 [Running DPDK Applications Without Root Privileges](https://doc.dpdk.org/guides/linux_gsg/enable_func.html#running-without-root-privileges)。

### 7.2.1. VFIO no-IOMMU 模式

如果系统上没有可用的 IOMMU，仍然可以使用 VFIO，但必须加载一个额外的模块参数：

```
modprobe vfio enable_unsafe_noiommu_mode=1
```

或者，也可以在已加载的内核模块中启用此选项：

```
echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
```

之后，VFIO 就可以像往常一样与硬件设备一起使用。

```
note: 
在使用enable_unsafe_noiommu_mode=1参数再次探测模块之前，可能需要卸载所有VFIO相关模块。
```

```
warn: 
由于 no-IOMMU 模式放弃了 IOMMU 保护，因此它本质上是不安全的。也就是说，在 IOMMU 不可用的情况下，它确实使用户能够保持 VFIO 所具有的设备访问和编程程度。
```

### 7.2.2. VFIO 内存映射限制

对于外部存储器或大页的 DMA 映射，使用 VFIO 接口。VFIO 不支持对曾经映射的内存进行部分取消映射。因此DPDK的内存以大页粒度或系统页粒度进行映射。DMA 映射的数量受到内核的限制，系统/大页内存的进程的用户锁定内存限制 (rlimit)。在内核 5.1 中添加了另一个适用于外部内存和系统内存的每个容器的总体限制，由 VFIO 模块参数 `dma_entry_limit` 定义，默认值为 64K。当应用程序超出 DMA 条目时，需要调整这些限制以增加允许的限制。

当使用`--no-huge`选项时，使用的页面大小是较小的4K或64K，我们需要增加`dma_entry_limit`。

要更新 `dma_entry_limit`，必须使用附加模块参数加载 `vfio_iommu_type1`：

```
modprobe vfio_iommu_type1 dma_entry_limit=512000
```

或者，也可以在已加载的内核模块中更改此值：

```
echo 512000 > /sys/module/vfio_iommu_type1/parameters/dma_entry_limit
```

### 7.2.3.使用 vfio-pci 创建虚拟功能

从Linux 5.7版本开始，`vfio-pci`模块支持创建虚拟功能。PF 绑定 `vfio-pci `模块后，用户可以使用 `sysfs` 接口创建 VFs，这些 VFs 将自动绑定到 `vfio-pci` 模块。

当 PF 绑定到 vfio-pci 时，默认情况下会有一个随机生成的 VF 令牌。出于安全原因，该令牌是只写的，因此用户无法直接从内核读取它。要访问 VF，用户需要创建一个新令牌，并使用它来初始化 VF 和 PF 设备。令牌采用 UUID 格式，因此任何 UUID 生成工具都可用于创建新令牌。

可以使用 EAL 参数 `--vfio-vf-token` 将该 VF 令牌传递给 DPDK。该令牌将用于应用程序内的所有 PF 和 VF 端口。

1. 通过uuid命令生成VF token

```
14d63f20-8445-11ea-8900-1f9ce7d5650d
```

2. 使用enable_sriov参数集加载vfio-pci模块

```
sudo modprobe vfio-pci enable_sriov=1
```

或者，如果模块已加载或内置，则通过 `sysfs` 传递 `enable_sriov` 参数：

```
echo 1 | sudo tee /sys/module/vfio_pci/parameters/enable_sriov
```

3. 将 PCI 设备绑定到 vfio-pci 驱动程序

```
./usertools/dpdk-devbind.py -b vfio-pci 0000:86:00.0
```

4. 创建所需数量的 VF 设备

```
echo 2 > /sys/bus/pci/devices/0000:86:00.0/sriov_numvfs
```

5. 启动将管理 PF 设备的 DPDK 应用程序

```
<build_dir>/app/dpdk-testpmd -l 22-25 -n 4 -a 86:00.0 \
--vfio-vf-token=14d63f20-8445-11ea-8900-1f9ce7d5650d --file-prefix=pf -- -i
```

6. 启动将管理 VF 设备的 DPDK 应用程序

```
<build_dir>/app/dpdk-testpmd -l 26-29 -n 4 -a 86:02.0 \
--vfio-vf-token=14d63f20-8445-11ea-8900-1f9ce7d5650d --file-prefix=vf0 -- -i
```

```
note: 
5.7 之前的Linux版本不支持在VFIO框架内创建虚拟功能。
```

### 7.2.4. VFIO 故障排除

在某些情况下，使用 `dpdk-devbind.py` 脚本将设备绑定到 VFIO 驱动程序可能会失败。首先要检查的是内核消息：

```
dmesg | tail
...
[ 1297.875090] vfio-pci: probe of 0000:31:00.0 failed with error -22
...
```

在大多数情况下，`error -22` 表示无法启用 VFIO 子系统，因为没有 IOMMU 支持

要检查内核是否已使用正确的参数启动，可以检查内核命令行：

```
cat /proc/cmdline
```

请参阅前面的部分，了解如何为您的系统正确配置内核参数。

如果内核配置正确，还必须确保 BIOS 配置具有虚拟化功能（例如 Intel® VT-d）。没有标准方法来检查平台是否配置正确，因此请检查您的平台文档以了解它是否具有此类功能以及如何启用它们。

在某些发行版中，默认内核配置是在编译时完全禁用 no-IOMMU 模式。可以在系统的引导配置中检查这一点：

```
cat /boot/config-$(uname -r) | grep NOIOMMU
# CONFIG_VFIO_NOIOMMU is not set
```

如果内核配置中未启用 `CONFIG_VFIO_NOIOMMU`，VFIO 驱动程序将不支持 no-IOMMU 模式，并且必须使用其他替代方案（例如 UIO 驱动程序）。

## 7.3. VFIO Platform

VFIO Platform 是一个内核驱动程序，通过添加对驻留在 IOMMU 后面的平台设备的支持来扩展 VFIO 的功能。Linux 通常在启动阶段直接从设备树中了解平台设备，这与内置必要信息的 PCI 设备不同。

要使用VFIO平台，必须首先加载vfio-platform模块：

```
sudo modprobe vfio-platform
```

``
note:
默认情况下，vfio-platform 假定平台设备具有专用的重置驱动程序。如果缺少此类驱动程序或设备不需要该驱动程序，可以通过设置 reset_required = 0 模块参数来关闭此选项。
``

之后平台设备需要绑定到`vfio-platform`。这是需要两个步骤的标准程序。首先，`driver_override` 在平台设备目录中可用，需要设置为 `vfio-platform`：

```
sudo echo vfio-platform > /sys/bus/platform/devices/DEV/driver_override
```

下一个 `DEV` 设备必须绑定到 `vfio-platform` 驱动程序：

```
sudo echo DEV > /sys/bus/platform/drivers/vfio-platform/bind
```

应用程序启动时，DPDK 平台总线驱动程序会扫描 `/sys/bus/platform/devices`，搜索具有指向 `vfio` 平台驱动程序的驱动程序符号链接的设备。最后，将扫描的设备与可用的 PMDs 进行匹配。

如果 PMD 名称或 PMD 别名与内核驱动程序名称匹配或 PMD 名称与平台设备名称匹配（全部按该顺序），则匹配成功。

VFIO 平台依赖于 ARM/ARM64，通常在这些系统上运行的发行版上启用。请查阅您的发行版文档以确保情况确实如此。

## 7.4.分流驱动器

使用分流驱动程序的 PMD 与设备内核驱动程序共存。在这种模型上，NIC 由内核控制，而数据路径由设备顶部的 PMD 直接执行。

这种模式有以下好处：
- 它是安全且健壮的，因为内存管理和隔离是由内核完成的。
- 它使用户能够在同一网络端口上运行 DPDK 应用程序时使用旧版 Linux 工具（例如 `ethtool` 或 `ifconfig`）。
- 它使 DPDK 应用程序能够仅过滤部分流量，而其余流量将由内核驱动程序引导和处理。流分叉由 NIC 硬件执行。例如，使用流隔离模式允许严格选择 DPDK 中接收的内容。

有关分流驱动程序的更多信息，请参阅 [NVIDIA bifurcated PMD](https://www.dpdk.org/wp-content/uploads/sites/35/2016/10/Day02-Session04-RonyEfraim-Userspace2016.pdf) 演示。

## 7.5. UIO

```
warn: 
使用 UIO 驱动程序本质上是不安全的，因为这种方法缺乏 IOMMU 保护，并且只能由 root 用户完成。
```

在无法使用 VFIO 的情况下，可以使用替代驱动程序。在许多情况下，Linux 内核中包含的标准 `uio_pci_generic` 模块可以用作 VFIO 的替代品。可以使用以下命令加载该模块：

```
sudo modprobe uio_pci_generic
```

```
note: 
uio_pci_generic 模块不支持创建虚函数。
```

作为 uio_pci_generic 的替代方案，可以在 [dpdk-kmods](http://git.dpdk.org/dpdk-kmods) 中找到 igb_uio 模块。可以如下图加载：

```
sudo modprobe uio
sudo insmod igb_uio.ko
```

```
note: 对于某些缺乏对传统中断支持的设备，例如虚拟功能 (VF) 设备，可能需要 `igb_uio` 模块来代替 `uio_pci_generic`。
```

```
note: 如果启用 UEFI 安全引导，Linux 内核可能不允许在系统上使用 UIO。因此，DPDK 使用的设备应绑定到 vfio-pci 内核模块，而不是任何基于 UIO 的模块。有关更多详细信息，请参阅下面的[ Binding and Unbinding Network Ports to/from the Kernel Modules](https://doc.dpdk.org/guides/linux_gsg/linux_drivers.html#linux-gsg-binding-kernel)。
```

```
note: 如果用于 DPDK 的设备绑定到基于 UIO 的内核模块，请确保 IOMMU 已禁用或处于直通模式。在 x86_64 系统上，可以在 GRUB 命令行中添加 intel_iommu=off 或 amd_iommu=off 或 intel_iommu=on iommu=pt ，或者在 aarch64 系统上添加 iommu.passthrough=1 。
```

