# 7. RX/TX 回调示例应用程序

RX/TX 回调示例应用程序是一个数据包转发应用程序，演示了如何在接收和传输的数据包上使用用户定义的回调。应用程序使用回调执行简单的延迟检查，以确定数据包在应用程序内花费的时间。


如果 NIC 支持硬件时间戳，则示例应用程序还将显示自数据包在硬件中添加时间戳以来的平均延迟，以及自 RX 回调接收和处理数据包以来的延迟。
（NIC 是网络接口卡（Network Interface Card）的缩写，也称为网络适配器或网卡。）

## 7.1.编译应用程序

要编译示例应用程序，请参阅 [Compiling the Sample Applications](https://doc.dpdk.org/guides/sample_app_ug/compiling.html)

该应用程序位于 `rxtx_callbacks` 子目录中。

## 7.2.运行应用程序

在linux环境下运行示例：

```bash
./<build_dir>/examples/dpdk-rxtx_callbacks -l 1 -n 4 -- [-t]
```

使用 -t 启用硬件时间戳。如果网卡不支持，将会显示错误。

有关运行应用程序和环境抽象层 (EAL) 选项的一般信息，请参阅 DPDK 入门指南。

### 7.3.1. Main 函数

main() 函数执行应用程序初始化并调用每个 lcore 的执行线程。该函数实际上与[基本转发示例应用程序](https://doc.dpdk.org/guides/sample_app_ug/skeleton.html)中解释的 main() 函数相同。

lcore_main() 函数也是相同的。

主要区别在于用户定义的 `port_init()` 函数，其中添加了回调。这将在下一节中解释：

### 7.3.2. port init函数

端口初始化的主要功能部分如下所示，并附有注释：

```c
static inline int
port_init(uint16_t port, struct rte_mempool *mbuf_pool)
{
	struct rte_eth_conf port_conf;
	const uint16_t rx_rings = 1, tx_rings = 1;
	uint16_t nb_rxd = RX_RING_SIZE;
	uint16_t nb_txd = TX_RING_SIZE;
	int retval;
	uint16_t q;
	struct rte_eth_dev_info dev_info;
	struct rte_eth_rxconf rxconf;
	struct rte_eth_txconf txconf;

	if (!rte_eth_dev_is_valid_port(port))
		return -1;

	memset(&port_conf, 0, sizeof(struct rte_eth_conf));

	retval = rte_eth_dev_info_get(port, &dev_info);
	if (retval != 0) {
		printf("Error during getting device (port %u) info: %s\n",
				port, strerror(-retval));

		return retval;
	}

	if (dev_info.tx_offload_capa & RTE_ETH_TX_OFFLOAD_MBUF_FAST_FREE)
		port_conf.txmode.offloads |=
			RTE_ETH_TX_OFFLOAD_MBUF_FAST_FREE;

	if (hw_timestamping) {
		if (!(dev_info.rx_offload_capa & RTE_ETH_RX_OFFLOAD_TIMESTAMP)) {
			printf("\nERROR: Port %u does not support hardware timestamping\n"
					, port);
			return -1;
		}
		port_conf.rxmode.offloads |= RTE_ETH_RX_OFFLOAD_TIMESTAMP;
		rte_mbuf_dyn_rx_timestamp_register(&hwts_dynfield_offset, NULL);
		if (hwts_dynfield_offset < 0) {
			printf("ERROR: Failed to register timestamp field\n");
			return -rte_errno;
		}
	}

	retval = rte_eth_dev_configure(port, rx_rings, tx_rings, &port_conf);
	if (retval != 0)
		return retval;

	retval = rte_eth_dev_adjust_nb_rx_tx_desc(port, &nb_rxd, &nb_txd);
	if (retval != 0)
		return retval;

	rxconf = dev_info.default_rxconf;

	for (q = 0; q < rx_rings; q++) {
		retval = rte_eth_rx_queue_setup(port, q, nb_rxd,
			rte_eth_dev_socket_id(port), &rxconf, mbuf_pool);
		if (retval < 0)
			return retval;
	}

	txconf = dev_info.default_txconf;
	txconf.offloads = port_conf.txmode.offloads;
	for (q = 0; q < tx_rings; q++) {
		retval = rte_eth_tx_queue_setup(port, q, nb_txd,
				rte_eth_dev_socket_id(port), &txconf);
		if (retval < 0)
			return retval;
	}

	retval  = rte_eth_dev_start(port);
	if (retval < 0)
		return retval;

	if (hw_timestamping && ticks_per_cycle_mult  == 0) {
		uint64_t cycles_base = rte_rdtsc();
		uint64_t ticks_base;
		retval = rte_eth_read_clock(port, &ticks_base);
		if (retval != 0)
			return retval;
		rte_delay_ms(100);
		uint64_t cycles = rte_rdtsc();
		uint64_t ticks;
		rte_eth_read_clock(port, &ticks);
		uint64_t c_freq = cycles - cycles_base;
		uint64_t t_freq = ticks - ticks_base;
		double freq_mult = (double)c_freq / t_freq;
		printf("TSC Freq ~= %" PRIu64
				"\nHW Freq ~= %" PRIu64
				"\nRatio : %f\n",
				c_freq * 10, t_freq * 10, freq_mult);
		/* TSC will be faster than internal ticks so freq_mult is > 0
		 * We convert the multiplication to an integer shift & mult
		 */
		ticks_per_cycle_mult = (1 << TICKS_PER_CYCLE_SHIFT) / freq_mult;
	}

	struct rte_ether_addr addr;

	retval = rte_eth_macaddr_get(port, &addr);
	if (retval < 0) {
		printf("Failed to get MAC address on port %u: %s\n",
			port, rte_strerror(-retval));
		return retval;
	}
	printf("Port %u MAC: %02"PRIx8" %02"PRIx8" %02"PRIx8
			" %02"PRIx8" %02"PRIx8" %02"PRIx8"\n",
			(unsigned)port,
			RTE_ETHER_ADDR_BYTES(&addr));

	retval = rte_eth_promiscuous_enable(port);
	if (retval != 0)
		return retval;

	/* RX and TX callbacks are added to the ports. 8< */
	rte_eth_add_rx_callback(port, 0, add_timestamps, NULL);
	rte_eth_add_tx_callback(port, 0, calc_latency, NULL);
	/* >8 End of RX and TX callbacks. */

	return 0;
}
```

RX 和 TX 回调作为函数指针添加到端口/队列中：
```c
rte_eth_add_rx_callback(port, 0, add_timestamps, NULL);
rte_eth_add_tx_callback(port, 0, calc_latency, NULL);
```

可以添加多个回调，并且可以将附加信息作为 `void*` 传递给回调函数指针。在上面的示例中使用了 `NULL`。

`add_timestamps()` 和 `calc_latency()` 函数解释如下。

### 7.3.3. add_timestamps() 回调

`add_timestamps()` 回调被添加到 RX 端口并应用于收到的所有数据包：

```c
static uint16_t
add_timestamps(uint16_t port __rte_unused, uint16_t qidx __rte_unused,
		struct rte_mbuf **pkts, uint16_t nb_pkts,
		uint16_t max_pkts __rte_unused, void *_ __rte_unused)
{
	unsigned i;
	uint64_t now = rte_rdtsc();

	for (i = 0; i < nb_pkts; i++)
		*tsc_field(pkts[i]) = now;
	return nb_pkts;
}
```

DPDK 函数 `rte_rdtsc()` 用于向每个数据包添加周期计数时间戳（有关详细信息，请参阅 DPDK API 文档的周期部分）

### 7.3.4. calc_latency() 回调

`calc_latency()` 回调被添加到 TX 端口，并在传输之前应用于所有数据包：

```c
static uint16_t
calc_latency(uint16_t port, uint16_t qidx __rte_unused,
		struct rte_mbuf **pkts, uint16_t nb_pkts, void *_ __rte_unused)
{
	uint64_t cycles = 0;
	uint64_t queue_ticks = 0;
	uint64_t now = rte_rdtsc();
	uint64_t ticks;
	unsigned i;

	if (hw_timestamping)
		rte_eth_read_clock(port, &ticks);

	for (i = 0; i < nb_pkts; i++) {
		cycles += now - *tsc_field(pkts[i]);
		if (hw_timestamping)
			queue_ticks += ticks - *hwts_field(pkts[i]);
	}

	latency_numbers.total_cycles += cycles;
	if (hw_timestamping)
		latency_numbers.total_queue_cycles += (queue_ticks
			* ticks_per_cycle_mult) >> TICKS_PER_CYCLE_SHIFT;

	latency_numbers.total_pkts += nb_pkts;

	if (latency_numbers.total_pkts > (100 * 1000 * 1000ULL)) {
		printf("Latency = %"PRIu64" cycles\n",
		latency_numbers.total_cycles / latency_numbers.total_pkts);
		if (hw_timestamping) {
			printf("Latency from HW = %"PRIu64" cycles\n",
			   latency_numbers.total_queue_cycles
			   / latency_numbers.total_pkts);
		}
		latency_numbers.total_cycles = 0;
		latency_numbers.total_queue_cycles = 0;
		latency_numbers.total_pkts = 0;
	}
	return nb_pkts;
}
```

`calc_latency()` 函数累计数据包总数和使用的总周期数。一旦传输的数据包超过 1 亿个，就会打印出每个数据包的平均周期计数并重置计数器。

