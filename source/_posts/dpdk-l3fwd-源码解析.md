---
title: dpdk_l3fwd-源码解析
date: 2017-05-10 10:21:43
tags:
categories: 
- DPDK
---

本文对DPDK17.02自带的example l3fwd进行走读解析。
该sample application基于DPDK进行3层包转发。该application展示了使用DPDK中hash库和LPM库来实现包转发。
初始化和runtime的过程很像[l2fwd][1]，主要区别就是l2fwd只是简单的将一个port的包全部转发到另一port,而l3fwd则是根据收到的包的information来决定转发到哪个port。
<!-- more -->

我们从`main()`函数开始进行分析。
首先调用``rte_eal_init(argc, argv)``进行EAL环境的初始化。
然后注册系统信号(signal()函数);
在然后设置每个port的目的mac address，存在全局数组`dest_eth_addr[]`中，l3fwd默认的将经过portid号port的包的目的mac address改为`02:00:00:00:00:00:portid`
```c
/* pre-init dst MACs for all ports to 02:00:00:00:00:xx */
	for (portid = 0; portid < RTE_MAX_ETHPORTS; portid++) {
		dest_eth_addr[portid] =
			ETHER_LOCAL_ADMIN_ADDR + ((uint64_t)portid << 40);
		*(uint64_t *)(val_eth + portid) = dest_eth_addr[portid];
	}
```

---
# parse_args(argc, argv)
然后调用`parse_args(argc, argv)`，解析l3fwd的参数。
```c
/* display usage */
static void
print_usage(const char *prgname)
{
	printf("%s [EAL options] --"
		" -p PORTMASK"
		" [-P]"
		" [-E]"
		" [-L]"
		" --config (port,queue,lcore)[,(port,queue,lcore)]"
		" [--eth-dest=X,MM:MM:MM:MM:MM:MM]"
		" [--enable-jumbo [--max-pkt-len PKTLEN]]"
		" [--no-numa]"
		" [--hash-entry-num]"
		" [--ipv6]"
		" [--parse-ptype]\n\n"

		"  -p PORTMASK: Hexadecimal bitmask of ports to configure\n"
		"  -P : Enable promiscuous mode\n"
		"  -E : Enable exact match\n"
		"  -L : Enable longest prefix match (default)\n"
		"  --config (port,queue,lcore): Rx queue configuration\n"
		"  --eth-dest=X,MM:MM:MM:MM:MM:MM: Ethernet destination for port X\n"
		"  --enable-jumbo: Enable jumbo frames\n"
		"  --max-pkt-len: Under the premise of enabling jumbo,\n"
		"                 maximum packet length in decimal (64-9600)\n"
		"  --no-numa: Disable numa awareness\n"
		"  --hash-entry-num: Specify the hash entry number in hexadecimal to be setup\n"
		"  --ipv6: Set if running ipv6 packets\n"
		"  --parse-ptype: Set to use software to analyze packet type\n\n",
		prgname);
}

```
在parse_args中
- `-p`: 设置enable_port_mask的值
- `-P`: 设置promiscuous_on，是否开启port的混杂模式
- `-E`: 设置l3fwd_em_on,  是否启用exact match
- `-L`: 设置l3fwd_lpm_on, 是否启用long prefix match
- `--config`: 调用`parse_config()`，这里只配置Rx 队列的信息
 + 将`(port,queue,lcore)`的映射关系存入lcore_params_array数组，最后把指针传给全局变量`lcore_params`
 + `(port,queue,lcore)`的映射表示port 的 queue号队列映射给lcore处理。
- `--eth-dest`: 调用`parse_eth_dest()`
 + parse_eth_dest()函数，根据参数，将参数传入的mac address存入`peer_addr[6]`数组
 + 然后用dest指针，指向dest_eth_addr,`dest = (uint8_t *)&dest_eth_addr[portid];`，
 + 在for循环中`dest[c] = peer_addr[c]`,从而配置了dest_eth_addr[]，以更改不同的port对应的目的mac address
- `--enable-jumbo`: 设置`port_conf.rxmode.jumbo_frame = 1`;
- `--max-pkt-len`:  设置`port_conf.rxmode.max_rx_pkt_len`;
- `--no-numa`: 设置`numa_on = 0`;即不启用numa,默认值为1
- `--ipv6`: 设置`ipv6 = 1`，即启用ipv6，默认值为0
- `--hash-entry-num`: 设置全局变量`hash_entry_number`，范围为[1,L3FWD_HASH_ENTRIES(1M)]
- `--parse-ptype`: 设置全局变量`parse_ptype = 1`，默认值为0;
 + 当为1时，即启用rx callback软件方式分析包的type
 + 当为0时，启用硬件分析。

 后面判断如果l3fwd_lpm_on和l3fwd_em_on同时都被置1，则退出；
 若都没被置1，则将l3fwd_lpm_on置1，表示默认使用long prefix match;
 然后若是l3fwd_lpm_on，则禁用ipv6,并将`hash_entery_number`置为默认值4，因为ipv6和hash只有在exact match时使用。

 ---
 调用`init_lcore_rx_queues()`函数，配置全局数组`lcore_conf[]`数组中每个具体的lcore所要处理的
- n_rx_queue接收队列数，
- 以及rx_queue_list[](即需要处理那个port的那个rx_queue);

```c
struct lcore_conf lcore_conf[RTE_MAX_LCORE];

struct lcore_rx_queue {
	uint8_t port_id;
	uint8_t queue_id;
} __rte_cache_aligned;

struct lcore_conf {
	uint16_t n_rx_queue;
	struct lcore_rx_queue rx_queue_list[MAX_RX_QUEUE_PER_LCORE];
	uint16_t n_tx_port;
	uint16_t tx_port_id[RTE_MAX_ETHPORTS];
	uint16_t tx_queue_id[RTE_MAX_ETHPORTS];
	struct mbuf_table tx_mbufs[RTE_MAX_ETHPORTS];
	void *ipv4_lookup_struct;
	void *ipv6_lookup_struct;
} __rte_cache_aligned;

```

调用`check_port_config(nb_ports)`检查`--config`配置的port是否被enabled_port_mask所允许和是否超过了nb_ports的限制。

---
# 建立查找表
然后运行`setup_l3fwd_lookup_tables()`，设置建立3层转发需要的转发查找表的函数调用。
根据全局变量l3fwd_em_on判断是采用exact match还是Long prefix match.
```c
static void
setup_l3fwd_lookup_tables(void)
{
	/* Setup HASH lookup functions. */
	if (l3fwd_em_on)
		l3fwd_lkp = l3fwd_em_lkp;
	/* Setup LPM lookup functions. */
	else
		l3fwd_lkp = l3fwd_lpm_lkp;
}
```

---
# 初始化各个ports
```c
/* initialize all ports */
	for (portid = 0; portid < nb_ports; portid++) {
		/* ...*/

		nb_rx_queue = get_port_n_rx_queues(portid);
		n_tx_queue = nb_lcores;
		if (n_tx_queue > MAX_TX_QUEUE_PER_PORT)
			n_tx_queue = MAX_TX_QUEUE_PER_PORT;
		printf("Creating queues: nb_rxq=%d nb_txq=%u... ",
			nb_rx_queue, (unsigned)n_tx_queue );
		ret = rte_eth_dev_configure(portid, nb_rx_queue,
					(uint16_t)n_tx_queue, &port_conf);
		if (ret < 0)
			rte_exit(EXIT_FAILURE,
				"Cannot configure device: err=%d, port=%d\n",
				ret, portid);

		rte_eth_macaddr_get(portid, &ports_eth_addr[portid]);
		print_ethaddr(" Address:", &ports_eth_addr[portid]);
		printf(", ");
		print_ethaddr("Destination:",
			(const struct ether_addr *)&dest_eth_addr[portid]);
		printf(", ");

		/*
		 * prepare src MACs for each port.
		 */
		ether_addr_copy(&ports_eth_addr[portid],
			(struct ether_addr *)(val_eth + portid) + 1);

		/* init memory */
		ret = init_mem(NB_MBUF);
		if (ret < 0)
			rte_exit(EXIT_FAILURE, "init_mem failed\n");

		/* init one TX queue per couple (lcore,port) */
		queueid = 0;
		for (lcore_id = 0; lcore_id < RTE_MAX_LCORE; lcore_id++) {
			if (rte_lcore_is_enabled(lcore_id) == 0)
				continue;

			if (numa_on)
				socketid =
				(uint8_t)rte_lcore_to_socket_id(lcore_id);
			else
				socketid = 0;

			printf("txq=%u,%d,%d ", lcore_id, queueid, socketid);
			fflush(stdout);

			rte_eth_dev_info_get(portid, &dev_info);
			txconf = &dev_info.default_txconf;
			if (port_conf.rxmode.jumbo_frame)
				txconf->txq_flags = 0;
			ret = rte_eth_tx_queue_setup(portid, queueid, nb_txd,
						     socketid, txconf);
			if (ret < 0)
				rte_exit(EXIT_FAILURE,
					"rte_eth_tx_queue_setup: err=%d, "
					"port=%d\n", ret, portid);

			qconf = &lcore_conf[lcore_id];
			qconf->tx_queue_id[portid] = queueid;
			queueid++;

			qconf->tx_port_id[qconf->n_tx_port] = portid;
			qconf->n_tx_port++;
		}
		printf("\n");
	}
```
- 首先调用`get_port_n_rx_queues(portid)`获取portid表示的port有几个rx queues;
- 设置`n_tx_queue = nb_lcores`;即为应用启动的时候EAL参数设置的用几个lcore;
- 然后调用`rte_eth_dev_configure(portid, nb_rx_queue,(uint16_t)n_tx_queue, &port_conf)`配置该port;
- 注意上一步最后传入的port_conf,在参数解析的时候，可能会设置`port_conf.rxmode.jumbo_frame`和`port_conf.rxmode.max_rx_pkt_len`;
**所以这里有几个lcore，对应就会给每个启用的port配置几个TX queues**
- 然后读取该port的MAC ADDRESS并输出，同时也输出该port的目的mac address，如果有用`--eth-dest`设置则为设置的值，否则为默认的值；
- 然后将该port的mac address 复制到val_eth结构里面，作为该port的源mac address存储。

然后调用init_mem(NB_MBUF)初始化内存，该函数中用for循环对每个启用的lcore进行处理，
- 首先根据是否启用了NUMA,来确定该lcore对应的NUMA节点号（socketid）
- 然后根据socketid，判断该socketid上的mempool是否创建了，已经存在的话，跳下一步；没有的话，则创建一个mempool
`pktmbuf_pool[socketid] = rte_pktmbuf_pool_create(s, nb_mbuf, MEMPOOL_CACHE_SIZE, 0, RTE_MBUF_DEFAULT_BUF_SIZE, socketid);`
并创建该socket上的查找表`l3fwd_lkp.setup(socketid)`
- 获取该lcore的配置结构体`qconf = &lcore_conf[lcore_id]`，分别指定`qconf->ipv4_lookup_struct`和`qconf->ipv6_lookup_struct`

然后初始化TX queue（有**几个lcore，就为每个port配置几个tx queue**）
- 首先根据是否启用NUMA,确定socketid
- 然后调用`rte_eth_dev_info_get(portid, &dev_info)`读取该port的设备信息;
- 调用`txconf = &dev_info.default_txconf`，获取tx的配置信息；
- 根据是否启用jumbo frame，配置txconf->txq_flags;
- 然后`rte_eth_tx_queue_setup(portid, queueid, nb_txd,socketid, txconf)`创建一个接收队列;
- 最后获取该lcore_id的lcore_conf[]配置数组，并更新tx相关的信息，
比如：`qconf->tx_queue_id[port]`表示该lcore处理portid的那个queue;
`qconf->n_tx_port`表示该lcore需要处理几个port的tx queue;
`qconf->tx_port_id[qconf->n_tx_port]`表示该lcore需要处理的tx queue的port_id;

然后初始化rx queue,for循环启用的lcores.
- 首先获取lcore的配置信息，`qconf = &lcore_conf[lcore_id]`
- 然后for循环根据配置文件中给这个lcore配置了需要处理几个rx queue,依次对每个rx queue进行配置
`for(queue = 0; queue < qconf->n_rx_queue; ++queue)`
 + 首先读取portid和queueid信息，
 + 然后根据是否启用numa_on，设置socketid
 + 然后调用`rte_eth_rx_queue_setup(portid, queueid, nb_rxd,socketid,NULL,pktmbuf_pool[socketid]);`建立tx queue

---
启动各个port和运行l3fwd包转发的死循环

1. 对各个启用的port调用`rte_eth_dev_start(portid)`，如果启用了混杂模式，则还需要调用`rte_eth_promiscuous_enable(portid)`
2. 对各个启用的lcore，先获取其配置数组`qconf= &lcore_conf[lcore_id]`，然后调用`prepare_ptype_parser(portid, queueid)`,验证该lcore需要处理的portid号port的queueid队列是否可能解析包的类型；
3. 调用`rte_eal_mp_remote_launch(l3fwd_lkp.main_loop, NULL, CALL_MASTER);`在每个lcore上启用l3fwd_lkp.main_loop死循环方法。

---
# 总结
整个l3fwd主要做了这几件事：
1. 首先调用`rte_eal_init(argc,argv)`，初始化EAL；
2. 调用`setup_l3fwd_lookup_tables（）`设置全局变量`l3fwd_lkp`是指向exact match结构还是long prefix match结构；
3. 然后在for循环中对每个enabled port调用`rte_eth_dev_configure(portid, nb_rx_queue,(uint16_t)n_tx_queue, &port_conf)`;
配置各个port,其中设置了rx queue和tx queue的数目，还有port的配置结构体port_conf;
4. 在同样的for循环中，对每个enabled port调用`init_mem()`，该函数其实和port没有关系，不知道为什么要放在port的循环中每次调用；
该函数里面对enabled lcore进行循环，做了2件事：
 + 判断该lcore属于的socket,是否已经创建了对应的`pktmbuf_pool`，没有则创建,并在创建的时候，调用`l3fwd_lkp.setup(socketid)`初始化查找表，可见查找表和socket相关；
 + 配置lcore的配置数组`lcore_conf[lcore_id]`的`ipv4_lookup_struct`和`ipv6_lookup_struct`
5. 在同样的for循环中，对每个enabled port,进行enable lcore循环，
 + 调用`rte_eth_dev_info_get(portid, &dev_info)`读取port的配置，然后根据读取的`dev_info`配置`txconf`,
 + 调用`rte_eth_tx_queue_setup(portid, queueid, nb_txd, socketid, txconf)`建立tx queue;
 + **说明：这里的2层循环是因为每个lcore会为每个port建立一个tx queue**
 + 最后会更新lcore的配置数组`lcore_conf[lcore_id]`
6. 在新的for循环中，对enabled lcore进行循环，根据lcore的配置数组`lcore_conf[lcore_id]`中的n_rx_queue进行循环，
 + 调用`rte_eth_rx_queue_setup(portid, queueid, nb_rxd, socketid, NULL, pktmbuf_pool[socketid])`,建立rx queue;
 + **说明：与第5步不同，每个port的rx queue得根据应用的启动时传入的参数`--config`配置的lcore_conf建立** 
7. 对每个enabled port进行循环调用`rte_eth_dev_start(portid)`和`rte_eth_promiscuous_enable(portid)`，分别启动每个port和设置port为混杂模式；
8. ptype_parser的处理；
9. 调用`rte_eal_mp_remote_launch(l3fwd_lkp.main_loop, NULL, CALL_MASTER)`，在每个enabled lcore上启动l3fwd_lkp.main_loop，准备对接收到的数据进行转发；
































[1]: https://calmisi.github.io/2017/05/09/dpdk-l2fwd-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/ "l2fwd源码解析"
