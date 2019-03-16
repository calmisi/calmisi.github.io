---
title: dpdk l2fwd 源码解析
date: 2017-05-09 16:50:35
tags:
categories: 
- DPDK
---

本文对DPDK17.02自带的example l2fwd进行走读解析。
<!-- more -->

从`main()`函数进行分析，
首先调用``rte_eal_init(argc, argv)``进行EAL环境的初始化，
然后执行
```c
argc -= ret;
argv += ret;
```
当初始化完EAL参数后，更新命令行参数，以便后面的参数被l2fwd这个特定的应用的参数解析函数`l2fwd_parse_args(argc, argv)`解析。

---
# 全局参数
```c
/* ethernet addresses of ports */
存储网络ports对应的mac address
static struct ether_addr l2fwd_ports_eth_addr[RTE_MAX_ETHPORTS];

/* mask of enabled ports */
启用的网络port的掩码，可以根据应用的参数设置
static uint32_t l2fwd_enabled_port_mask = 0;

/* list of enabled ports */
存储不同的网络ports的转发port,即port 0转到port 1
static uint32_t l2fwd_dst_ports[RTE_MAX_ETHPORTS];

设置一个lcore处理几个rx队列
static unsigned int l2fwd_rx_queue_per_lcore = 1;
```

---
# l2fwd_parse_args
```c
/* display usage */
static void
l2fwd_usage(const char *prgname)
{
	printf("%s [EAL options] -- -p PORTMASK [-q NQ]\n"
	       "  -p PORTMASK: hexadecimal bitmask of ports to configure\n"
	       "  -q NQ: number of queue (=ports) per lcore (default is 1)\n"
		   "  -T PERIOD: statistics will be refreshed each PERIOD seconds (0 to disable, 10 default, 86400 maximum)\n"
		   "  --[no-]mac-updating: Enable or disable MAC addresses updating (enabled by default)\n"
		   "      When enabled:\n"
		   "       - The source MAC address is replaced by the TX port MAC address\n"
		   "       - The destination MAC address is replaced by 02:00:00:00:00:TX_PORT_ID\n",
	       prgname);
}
```
该函数给出了l2fwd支持的参数说明。

下面来看`l2fwd_parse_args(argc, argv)`函数，
1. 首先调用getopt_long解析`--`开始的长参数，即`--no-mac-updating`或`--mac-updating`，来设置`mac_updating`的值，后面根据这个全局变量的值来决定转发的时候是否更新mac address;
1. `-p`: 将需要使用的ports的掩码存在`l2fwd_enabled_port_mask`里面, 默认值为0；
2. `-q`: 设置`l2fwd_rx_queue_per_lcore`的值，表示一个lcore可以处理几个rx queue， 默认值为1；
3. `-T`: 设置`timer_period`，表示statistics timer_period秒输出一次

---
后面调用`rte_pktmbuf_pool_create()`创建该应用使用的内存池mempool;

然后执行
```c
/* reset l2fwd_dst_ports */
  首先重置所有port的目的转发port
	for (portid = 0; portid < RTE_MAX_ETHPORTS; portid++)
		l2fwd_dst_ports[portid] = 0;
	last_port = 0;

  依次两两成对的组成转发pair
	for (portid = 0; portid < nb_ports; portid++) {
		/* skip ports that are not enabled */
		if ((l2fwd_enabled_port_mask & (1 << portid)) == 0)
			continue;

		if (nb_ports_in_mask % 2) {
			l2fwd_dst_ports[portid] = last_port;
			l2fwd_dst_ports[last_port] = portid;
		}
		else
			last_port = portid;

		nb_ports_in_mask++;

		rte_eth_dev_info_get(portid, &dev_info);
	}
  当l2fwd_enabled_port_mask掩码所表示的ports数目（nb_ports_in_mask）为奇数时
	if (nb_ports_in_mask % 2) {
		printf("Notice: odd number of ports in portmask.\n");
		l2fwd_dst_ports[last_port] = last_port;
	}
```
设置启用的网络ports对应的转发port，该l2fwd应用应该启用偶数个ports,一次两两组成pair，相互转发数据；

---

# 根据`l2fwd_rx_queue_per_lcore`设置`lcore_queue_conf`数组，即每个lcore可以处理几个rx队列
```c
  rx_lcore_id = 0;
  qconf = NULL;

/* Initialize the port/queue configuration of each logical core */
	for (portid = 0; portid < nb_ports; portid++) {
		/* skip ports that are not enabled */
		if ((l2fwd_enabled_port_mask & (1 << portid)) == 0)
			continue;

		/* get the lcore_id for this port */
    // rx_lcore_id从0开始
    // 如果rx_lcore_id表示的lcore没有被启用，
    // 或者该lcore对应的lcore_queue_conf数组的n_rx_port(分配的要处理的rx port的数目)已经达到了设置的l2fwd_rx_queue_per_lcore
    // 则检查下一个lcore

		while (rte_lcore_is_enabled(rx_lcore_id) == 0 ||
		       lcore_queue_conf[rx_lcore_id].n_rx_port ==
		       l2fwd_rx_queue_per_lcore) {
			rx_lcore_id++;
			if (rx_lcore_id >= RTE_MAX_LCORE)
				rte_exit(EXIT_FAILURE, "Not enough cores\n");
		}

		if (qconf != &lcore_queue_conf[rx_lcore_id])
			/* Assigned a new logical core in the loop above. */
			qconf = &lcore_queue_conf[rx_lcore_id];

		qconf->rx_port_list[qconf->n_rx_port] = portid;
		qconf->n_rx_port++;
		printf("Lcore %u: RX port %u\n", rx_lcore_id, (unsigned) portid);
	}
```

```c
struct lcore_queue_conf {
	unsigned n_rx_port;
	unsigned rx_port_list[MAX_RX_QUEUE_PER_LCORE];
} __rte_cache_aligned;
struct lcore_queue_conf lcore_queue_conf[RTE_MAX_LCORE];
```
`lcore_queue_conf`结构体数组，存储对应的lcore的分配情况，
比如lcore_queue_conf[i]表示第i号lcore总共需要处理lcore_queue_conf[i].n_rx_port个RX 网络port,
并且这n_rx_port个网络port的编号存在lcore_queue_conf[i].rx_port_list数组里面；

---

# 初始化所有enabled ports
```c
/* Initialise each port */
	for (portid = 0; portid < nb_ports; portid++) {
		/* skip ports that are not enabled */
		if ((l2fwd_enabled_port_mask & (1 << portid)) == 0) {
			printf("Skipping disabled port %u\n", (unsigned) portid);
			nb_ports_available--;
			continue;
		}
		/* init port */
		printf("Initializing port %u... ", (unsigned) portid);
		fflush(stdout);
		ret = rte_eth_dev_configure(portid, 1, 1, &port_conf);
		if (ret < 0)
			rte_exit(EXIT_FAILURE, "Cannot configure device: err=%d, port=%u\n",
				  ret, (unsigned) portid);

		rte_eth_macaddr_get(portid,&l2fwd_ports_eth_addr[portid]);

		/* init one RX queue */
		fflush(stdout);
		ret = rte_eth_rx_queue_setup(portid, 0, nb_rxd,
					     rte_eth_dev_socket_id(portid),
					     NULL,
					     l2fwd_pktmbuf_pool);
		if (ret < 0)
			rte_exit(EXIT_FAILURE, "rte_eth_rx_queue_setup:err=%d, port=%u\n",
				  ret, (unsigned) portid);

		/* init one TX queue on each port */
		fflush(stdout);
		ret = rte_eth_tx_queue_setup(portid, 0, nb_txd,
				rte_eth_dev_socket_id(portid),
				NULL);
		if (ret < 0)
			rte_exit(EXIT_FAILURE, "rte_eth_tx_queue_setup:err=%d, port=%u\n",
				ret, (unsigned) portid);

		/* Initialize TX buffers */
		tx_buffer[portid] = rte_zmalloc_socket("tx_buffer",
				RTE_ETH_TX_BUFFER_SIZE(MAX_PKT_BURST), 0,
				rte_eth_dev_socket_id(portid));
		if (tx_buffer[portid] == NULL)
			rte_exit(EXIT_FAILURE, "Cannot allocate buffer for tx on port %u\n",
					(unsigned) portid);

		rte_eth_tx_buffer_init(tx_buffer[portid], MAX_PKT_BURST);

		ret = rte_eth_tx_buffer_set_err_callback(tx_buffer[portid],
				rte_eth_tx_buffer_count_callback,
				&port_statistics[portid].dropped);
		if (ret < 0)
				rte_exit(EXIT_FAILURE, "Cannot set error callback for "
						"tx buffer on port %u\n", (unsigned) portid);

		/* Start device */
		ret = rte_eth_dev_start(portid);
		if (ret < 0)
			rte_exit(EXIT_FAILURE, "rte_eth_dev_start:err=%d, port=%u\n",
				  ret, (unsigned) portid);

		printf("done: \n");

		rte_eth_promiscuous_enable(portid);

		printf("Port %u, MAC address: %02X:%02X:%02X:%02X:%02X:%02X\n\n",
				(unsigned) portid,
				l2fwd_ports_eth_addr[portid].addr_bytes[0],
				l2fwd_ports_eth_addr[portid].addr_bytes[1],
				l2fwd_ports_eth_addr[portid].addr_bytes[2],
				l2fwd_ports_eth_addr[portid].addr_bytes[3],
				l2fwd_ports_eth_addr[portid].addr_bytes[4],
				l2fwd_ports_eth_addr[portid].addr_bytes[5]);

		/* initialize port stats */
		memset(&port_statistics, 0, sizeof(port_statistics));
	}
```
以上代码，在for循环中初始化每个网络port.
首先根据`l2fwd_enabled_port_mask`来判断本次循环所要操作的portid是否被enabled，没有的话，就更新`nb_ports_available`的值，并进入下一次循环；
否则就初始化该portid所表示的网卡port,

1. 首先调用`rte_eth_dev_configure(portid, 1, 1, &port_conf)`配置该portid的网卡port,
其中第二2个参数是nb_rx_queue,第三个参数是nb_tx_queue，
即配置该网卡一个rx队列和一个tx队列。

2. 配置完成之后，调用`rte_eth_macaddr_get(portid,&l2fwd_ports_eth_addr[portid])`将对应portid的port mac address读入之前定义的全局变量l2fwd_ports_eth_addr[portid];

3. 然后调用`rte_eth_rx_queue_setup(portid, 0, nb_rxd, rte_eth_dev_socket_id(portid), NULL, l2fwd_pktmbuf_pool);`
从l2fwd_pktmbuf_pool内存池（main函数申请，见上面）中为receive ring申请nb_rxd个receive desciptor，
其中第二个参数0表示该rx queue的rx_queue_id（该值属于[0, nb_rx_queue -1], nb_rx_queue即为第一步中rte_eth_dev_configure中配置的rx queue的数目，这里为1）；
其中第5个参数为`const struct rte_eth_rxconf *rx_conf`,这里传入的为NULL，即不配置。

4. 然后调用`rte_eth_tx_queue_setup(portid, 0, nb_txd,
				rte_eth_dev_socket_id(portid),
				NULL);`
注意参数基本与上一步相同，只是最后一个参数没有加pktmbuf_pool,为什么呢，请看第5步

5. 初始化Tx buffers
```
/* Initialize TX buffers */
		tx_buffer[portid] = rte_zmalloc_socket("tx_buffer",
				RTE_ETH_TX_BUFFER_SIZE(MAX_PKT_BURST), 0,
				rte_eth_dev_socket_id(portid));
		if (tx_buffer[portid] == NULL)
			rte_exit(EXIT_FAILURE, "Cannot allocate buffer for tx on port %u\n",
					(unsigned) portid);

		rte_eth_tx_buffer_init(tx_buffer[portid], MAX_PKT_BURST);

		ret = rte_eth_tx_buffer_set_err_callback(tx_buffer[portid],
				rte_eth_tx_buffer_count_callback,
				&port_statistics[portid].dropped);
		if (ret < 0)
				rte_exit(EXIT_FAILURE, "Cannot set error callback for "
						"tx buffer on port %u\n", (unsigned) portid);
```
 - 其中tx_buffer为定义为全局变量`static struct rte_eth_dev_tx_buffer *tx_buffer[RTE_MAX_ETHPORTS];`
利用`void * rte_zmalloc_socket(const char *type, size_t size, unsigned align, int socket);`为该portid动态申请zero填充的空间，
注意申请的size为`RTE_ETH_TX_BUFFER_SIZE(MAX_PKT_BURST)`,该宏表示的大小为 MAX_PKT_BURST * sizeof(struct rte_mbuf *) + sizeof(struct rte_eth_dev_tx_buffer)即MAX_PKT_BURST个ret_mbuf的指针(存放要发送的包)，和一个自己指向的结构体的大小(tx_buffer[i] 为struct rte_eth_dev_tx_buffer * 的指针)。

 - 然后调用`rte_eth_tx_buffer_init(tx_buffer[portid], MAX_PKT_BURST);`来初始化上一步申请的tx_buffer;

 - 然后设置错误处理回调函数

6. 调用`rte_eth_dev_start(portid)`来启动该网卡port，并设置为混杂模式，输出该prot的mac address,并将port的统计数据置0

---
然后调用`check_all_ports_link_status(nb_ports, l2fwd_enabled_port_mask);`检查每个port的状态，
最后调用`rte_eal_mp_remote_launch(l2fwd_launch_one_lcore, NULL, CALL_MASTER)`在所有核上调用`l2fwd_launch_one_lcore（）`函数(包括主lcore)

当lcore退出l2fwd_launch_one_lcore()死循环函数后，就检查其他lcore的状态，置ret，
最后for循环stop,close各个port
程序就退出了。

---
# l2fwd_launch_one_lcore()死循环处理函数
下面重点来看``
