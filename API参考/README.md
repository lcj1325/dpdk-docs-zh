# API

公共 API 标头按主题分组：

- device: dev, ethdev, ethctrl, rte_flow, rte_tm, rte_mtr, bbdev, cryptodev, security, compressdev, compress, regexdev, mldev, dmadev, gpudev, eventdev, event_eth_rx_adapter, event_eth_tx_adapter, event_timer_adapter, event_crypto_adapter, event_dma_adapter, rawdev, metrics, bitrate, latency, devargs, PCI, vdev, vfio
- device specific: softnic, bond, vhost, vdpa, ixgbe, i40e, iavf, bnxt, cnxk, cnxk_crypto, cnxk_eventdev, cnxk_mempool, dpaa, dpaa2, mlx5, dpaa2_mempool, dpaa2_cmdif, dpaa2_qdma, crypto_scheduler, dlb2, ifpga
- memory: memseg, memzone, mempool, malloc, memcpy
- timers: cycles, timer, alarm
- locks: atomic, mcslock, pflock, rwlock, seqcount, seqlock, spinlock, ticketlock, RCU
- CPU arch: branch prediction, cache prefetch, SIMD, byte order, CPU flags, CPU pause, I/O access, power management
- CPU multicore: interrupts, launch, lcore, per-lcore, service cores, keepalive, power/freq, PMD power
- layers: ethernet, MACsec, ARP, HIGIG, ICMP, ESP, IPsec, IPsec group, IPsec SA, IPsec SAD, IP, frag/reass, UDP, SCTP, TCP, TLS, DTLS, GTP, GRO, GSO, GRE, MPLS, VXLAN, Geneve, eCPRI, PDCP hdr, PDCP, L2TPv2, PPP, IB
- QoS: metering, scheduler, RED congestion
- routing: LPM IPv4 route, LPM IPv6 route, RIB IPv4, RIB IPv6, FIB IPv4, FIB IPv6
- hashes: hash, jhash, thash, thash_gfni, FBK hash, CRC hash
- classification reorder, dispatcher, distributor, EFD, ACL, member, BPF
- containers: mbuf, mbuf pool ops, ring, stack, tailq, bitmap
- packet framework:
  - port: ethdev, ring, frag, reass, sched, src/sink
  - table: lpm IPv4, lpm IPv6, ACL, hash, array, stub
  - pipeline port_in_action table_action
  - SWX pipeline: control, extern, pipeline
  - SWX port: port, ethdev, fd, ring, src/sink
  - SWX table: table, table_em table_wm
  - graph: graph_worker
  - graph_nodes: eth_node, ip4_node, ip6_node, udp4_input_node
- basic: bitops, approx fraction, random, config file, key/value args, argument parsing, string, thread
- debug: jobstats, telemetry, pcapng, pdump, hexdump, debug, log, errno, trace, trace_point
- misc: EAL config, common, experimental APIs, ABI versioning, version