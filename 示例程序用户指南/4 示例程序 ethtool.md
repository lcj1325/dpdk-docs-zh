
# 4. Ethtool 示例应用程序

Ethtool 示例应用程序展示了类似 ethtool 的 API 的实现，并提供了一个控制台环境，允许使用它来查询和更改以太网卡参数。该示例基于简单的 L2 框架反射器。

# 4.1.编译应用程序

要编译示例应用程序，请参阅 [Compiling the Sample Applications](https://doc.dpdk.org/guides/sample_app_ug/compiling.html)

该应用程序位于 `ethtool` 子目录中。

# 4.2.运行应用程序

该应用程序需要每个端口有一个可用内核，再加上一个。唯一可用的选项是 EAL 的标准选项：

```
./<build_dir>/examples/dpdk-ethtool [EAL options]
```

有关运行应用程序和环境抽象层 (EAL, Environment Abstraction Layer) 选项的一般信息，请参阅 DPDK *Getting Started Guide*。

## 4.3.使用应用程序

该应用程序是使用 cmdline DPDK 界面进行控制台驱动的：

```
EthApp>
```

在此界面中，可用命令及其功能描述如下：
- `drvinfo`：打印驱动程序信息
- `eeprom`：将 EEPROM 转储到文件
- `module-eeprom`：将插件模块 EEPROM 转储到文件
- `link`：打印端口链接状态
- `macaddr`：获取/设置MAC地址
- `mtu`：设置网卡MTU
- `open`：开放端口
- `pause`：获取/设置端口暂停状态
- `portstats`：打印端口统计信息
- `regs`：将端口寄存器转储到文件
- `ringparam`：获取/设置环参数
- `rxmode`：切换端口接收模式
- `stop`：停止端口
- `validate`：检查给定的 MAC 地址是否是有效的单播地址
- `vlan`：添加/删除 VLAN id
- `quit`：退出程序

4.4.解释

示例程序有两部分：在工作核心上运行的后台 [packet reflector](https://doc.dpdk.org/guides/sample_app_ug/ethtool.html#packet-reflector) 和在主核心上运行的前台 [Ethtool Shell](https://doc.dpdk.org/guides/sample_app_ug/ethtool.html#ethtool-shell)。这些将在下面描述。

### 4.4.1.数据包反射器

后台数据包反射器旨在演示由 Ethtool shim 控制的 NIC 端口上的基本数据包处理。每个传入的 MAC 帧都会被重写，以便使用相关端口自己的 MAC 地址作为源地址将其返回给发送方，然后在同一端口上发送出去。

### 4.4.2.以太网工具外壳

Ethtool 示例的前台部分是一个基于控制台的界面，它接受 [using the application](https://doc.dpdk.org/guides/sample_app_ug/ethtool.html#using-the-application) 中所述的命令。各个回调函数处理与每个命令相关的详细信息，这些函数使用 DPDK 函数的 [Ethtool interface](https://doc.dpdk.org/guides/sample_app_ug/ethtool.html#ethtool-interface) 中定义的函数。

## 4.5.以太网工具接口

Ethtool接口作为一个单独的库构建，并实现以下功能：

- rte_ethtool_get_drvinfo()
- rte_ethtool_get_regs_len()
- rte_ethtool_get_regs()
- rte_ethtool_get_link()
- rte_ethtool_get_eeprom_len()
- rte_ethtool_get_eeprom()
- rte_ethtool_set_eeprom()
- rte_ethtool_get_module_info()
- rte_ethtool_get_module_eeprom()
- rte_ethtool_get_pauseparam()
- rte_ethtool_set_pauseparam()
- rte_ethtool_net_open()
- rte_ethtool_net_stop()
- rte_ethtool_net_get_mac_addr()
- rte_ethtool_net_set_mac_addr()
- rte_ethtool_net_validate_addr()
- rte_ethtool_net_change_mtu()
- rte_ethtool_net_get_stats64()
- rte_ethtool_net_vlan_rx_add_vid()
- rte_ethtool_net_vlan_rx_kill_vid()
- rte_ethtool_net_set_rx_mode()
- rte_ethtool_get_ringparam()
- rte_ethtool_set_ringparam()
