---
layout: post
title:  "skb的生命周期"
date:   2017-05-05
categories: linux kernel
---

*本文基于 linux 3.10.105*

`skb` 指的是 `sk_buff` 这个结构体

#### 网卡驱动的一般流程

这里以 `intel e1000` 驱动举例, 

* 注册 PCI (PCI-E) 设备, 挂接 PCI 硬件基本操作接口, 最关键的是 `probe`

  注册 PCI 设备时, 系统从 PCI 总线上读取 deviceId 和 vendorId, 来确认是那家公司的什么设备, 比如这里 Intel 的 venderId 就是 `0x8086` , 参考 `struct pci_device_id`

* 系统找到对应注册的 PCI 设备, 初始化改设备时, 调用 probe 这个函数

* 挂接网络设备处理函数 `net_device->netdev_ops`, 包括: up/down/设置mac/mtu 等等

* 调用 `netif_napi_add` 注册轮询函数 `e1000_clean`  加入 NAPI 框架

* `e1000_open` 初始化软硬件, 包括收发队列, skb 申请, DMA 映射等

* 系统根据设置或者用户操作, up 该 NIC, 调用 ndo_open, 驱动挂接中断向量

* 网卡收到包或者发送完成时, 向 CPU 捅中断, 调用对应的中断处理函数, 也就是上一步挂接的函数 `e1000_intr`

* 关网卡中断, 交给 NAPI 框架处理

* NAPI 框架调用到 `e1000_clean`  , 清理写(发包) 和 处理读 (收包)

#### 收包的skb

从上面的流程可以看出

* skb 是在设备 up 的时候申请的, 并调用 `dma_map_single` 用 `skb->data` 建立一个用于单次操作的流式 DMA 映射
* 获取到物理地址 (buffer_info->dma) 后, 挂接到网卡的收包队列中
* 网卡收到包之后, 报文内容被写入到 `skb->data` , 给 CPU 中断
* NAPI 接管之后调用驱动轮询函数接收报文
* 驱动设置好 `skb->protocol`
* 调用 `napi_gro_receive`, 走 NAPI 统一流程

#### NAPI

在 NAPI 之前, Linux 网卡收包完全依赖于网卡中断, 大概的模式就是

> 网卡: 诶, 收到一个包, 给 CPU 捅一个中断
>
> 内核: 从网卡拿一个包

在这个模式下, 应对持续大流量就比较费劲儿, CPU 都用来处理中断了.

NAPI 建立了一个新的框架, 要做的事儿就是 (和路由器的收包流程类似)

> 收到一个中断之后, 通知驱动关网卡中断, 不让中断再来烦 CPU 了, 使用轮询模式把网卡的收包队列读读读, 一直读空, 再开接收中断

*(intel 的 DPDK 做的更绝, 直接不要中断了, 死循环去读)*

NAPI 效果的实现, 依赖于驱动对应的实现, 驱动需要响应框架的 `关中断/轮询收` 命令

#### GRO

网卡收到的包有时是和前后的包关联的 (比如 TCP 的分段报文), 内核协议栈处理报文时只用到报文的头部, 如果把这些相关报文合并, 那么内核协议栈处理就会更快了. GRO 的目的就是将这些报文合并.



#### skb 从哪里来

##### 结构体说明

```c
struct sk_buff {
    /* These two members must be first. */
    struct sk_buff      *next;  /* 和 prev 用来维护 skb 链表 */
    struct sk_buff      *prev;

    ktime_t             tstamp; /* 接收或者发送时间戳 */

    struct sock         *sk; /* 指向对应套接字 */
    struct net_device   *dev; /* 指向接收和发送设备, 在生命周期可能会变 */

    /*
     * This is the control buffer. It is free to use for every
     * layer. Please put your private variables there. If you
     * want to keep them across layers you have to do a skb_clone()
     * first. This is owned by whoever has the skb queued ATM.
     */
    char                cb[48] __aligned(8);

    unsigned long       _skb_refdst;
#ifdef CONFIG_XFRM
    struct  sec_path    *sp;
#endif
    unsigned int        len, /* skb 管理的数据的总长度 */
                        data_len; /* skb 管理的非线性数据的长度, 所以 skb_headlen = skb->len - skb->data_len */
    __u16               mac_len, /* mac头长度 */
                        hdr_len;
    union {
         __wsum         csum;
        struct {
            __u16       csum_start;
            __u16       csum_offset;
        };
    };
    __u32               priority;
    kmemcheck_bitfield_begin(flags1);
    __u8                local_df:1,
                        cloned:1,
                        ip_summed:2,
                        nohdr:1,
                        nfctinfo:3;
    __u8                pkt_type:3,
                        fclone:2,
                        ipvs_property:1,
                        peeked:1,
                        nf_trace:1;
    kmemcheck_bitfield_end(flags1);
    __be16              protocol; /* L3协议,  <include/uapi/linux/if_ether.h>*/

    void                (*destructor)(struct sk_buff *skb);
#if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)
    struct nf_conntrack *nfct;
#endif
#ifdef CONFIG_BRIDGE_NETFILTER
    struct nf_bridge_info   *nf_bridge;
#endif

    int                 skb_iif; /* 接口设备ifIndex */

    __u32               rxhash;

    __be16              vlan_proto;
    __u16               vlan_tci;

#ifdef CONFIG_NET_SCHED
    __u16               tc_index;   /* traffic control index */
#ifdef CONFIG_NET_CLS_ACT
    __u16               tc_verd;    /* traffic control verdict */
#endif
#endif

    __u16               queue_mapping;
    kmemcheck_bitfield_begin(flags2);
#ifdef CONFIG_IPV6_NDISC_NODETYPE
    __u8                ndisc_nodetype:2;
#endif
    __u8                pfmemalloc:1;
    __u8                ooo_okay:1;
    __u8                l4_rxhash:1;
    __u8                wifi_acked_valid:1;
    __u8                wifi_acked:1;
    __u8                no_fcs:1;
    __u8                head_frag:1;
    /* Encapsulation protocol and NIC drivers should use
     * this flag to indicate to each other if the skb contains
     * encapsulated packet or not and maybe use the inner packet
     * headers if needed
     */
    __u8                encapsulation:1;  /* 是否用于封装报文, 比如 vxlan, 这时就需要后面的inner开头的header */
    /* 7/9 bit hole (depending on ndisc_nodetype presence) */
    kmemcheck_bitfield_end(flags2);

#ifdef CONFIG_NET_DMA
    dma_cookie_t        dma_cookie;
#endif
#ifdef CONFIG_NETWORK_SECMARK
    __u32               secmark;
#endif
    union {
        __u32           mark;
        __u32           dropcount;
        __u32           reserved_tailroom;
    };

    sk_buff_data_t      inner_transport_header; /* encapsulation 为1时的传输层头 */
    sk_buff_data_t      inner_network_header; /* encapsulation 为1时的网络层头 */
    sk_buff_data_t      inner_mac_header; /* encapsulation 为1时的链路层头 */
    sk_buff_data_t      transport_header; /* 传输层的头 */
    sk_buff_data_t      network_header; /* 网络层头 */
    sk_buff_data_t      mac_header; /* 链路层的头 */
    /* These elements must be at the end, see alloc_skb() for details.  */
    sk_buff_data_t      tail; /* 线性 buff 有效数据的结束 */
    sk_buff_data_t      end; /* 线性 buff 的实际结束 */
    unsigned char       *head, /* 线性 buff 的实际开始 */
                        *data; /* 线性 buff 有效数据的开始 */
    unsigned int        truesize; /* 本 sk_buff + skb_shared_info 占用长度  */
    atomic_t            users; /* skb 引用计数 */
};
```



##### skb 的申请

```c
static inline struct sk_buff *netdev_alloc_skb_ip_align(struct net_device *dev, unsigned int length)
{
	return __netdev_alloc_skb_ip_align(dev, length, GFP_ATOMIC); // 加 GFP_ATOMIC 调用
}

// 加偏移 NET_IP_ALIGN = 2
static inline struct sk_buff *__netdev_alloc_skb_ip_align(struct net_device *dev,
		unsigned int length, gfp_t gfp)
{
	struct sk_buff *skb = __netdev_alloc_skb(dev, length + NET_IP_ALIGN, gfp);

	if (NET_IP_ALIGN && skb)
		skb_reserve(skb, NET_IP_ALIGN);
	return skb;
}

struct sk_buff *__netdev_alloc_skb(struct net_device *dev,
				   unsigned int length, gfp_t gfp_mask)
{
	struct sk_buff *skb = NULL;
    // 计算需要的空间(包括skb_shared_info)
	unsigned int fragsz = SKB_DATA_ALIGN(length + NET_SKB_PAD) +
			      SKB_DATA_ALIGN(sizeof(struct skb_shared_info));
    // 如果 cache 够用, 直接在 cache 里分配
	if (fragsz <= PAGE_SIZE && !(gfp_mask & (__GFP_WAIT | GFP_DMA))) {
		void *data;

		if (sk_memalloc_socks())
			gfp_mask |= __GFP_MEMALLOC;
        // 从 per cpu 的 netdev_alloc_cache 里分配 data
		data = __netdev_alloc_frag(fragsz, gfp_mask);

		if (likely(data)) {
            // 从 skbuff_head_cache 里分配/初始化 sk_buff 结构体, 并挂接 data
			skb = build_skb(data, fragsz);
			if (unlikely(!skb))
				put_page(virt_to_head_page(data));
		}
	} else {
        // 普通分配内存并创建skb流程
		skb = __alloc_skb(length + NET_SKB_PAD, gfp_mask,
				  SKB_ALLOC_RX, NUMA_NO_NODE);
	}
	if (likely(skb)) {
		skb_reserve(skb, NET_SKB_PAD);
		skb->dev = dev;
	}
	return skb;
}
```

* GFP_ATOMIC

  平时主要用的是 `GFP_ATOMIC` 和 `GFP_KERNEL` 两个, 区别如下

  * 在使用 `GFP_KERNEL ` 标志申请内存时, 如果内存不够, 当前进程可能会睡眠等待足够的内存, 所以当前进程必须是可重入的, 所以在中断上下文不可能用这个标志
  * 在使用 `GFP_KERNEL ` 标志申请内存时, 如果内存不够, 直接返回失败, 不会睡眠

* NET_IP_ALIGN 偏移

  为什么要用几个函数来包装一层, 就是为了一个 2 字节的偏移? 

  > Since an ethernet header is 14 bytes network drivers often end up with  the IP header at an unaligned offset. The IP header can be aligned by shifting the start of the packet by 2 bytes

* NET_SKB_PAD 偏移

  > The networking layer reserves some headroom in skb data (via dev_alloc_skb). This is used to avoid having to reallocate skb data when the header has to grow. In the default case, if the header has to grow 32 bytes or less we avoid the reallocation.



#### 经过哪里

##### GRO 统一收包

```c
gro_result_t napi_gro_receive(struct napi_struct *napi, struct sk_buff *skb)
{
	skb_gro_reset_offset(skb);
    // 组包
	return napi_skb_finish(dev_gro_receive(napi, skb), skb);
}

static gro_result_t napi_skb_finish(gro_result_t ret, struct sk_buff *skb)
{
	switch (ret) {
	case GRO_NORMAL:
        // 组完之后调用 netif_receive_skb
		if (netif_receive_skb(skb))
			ret = GRO_DROP;
		break;

	case GRO_DROP:
		kfree_skb(skb);
		break;

	case GRO_MERGED_FREE:
		if (NAPI_GRO_CB(skb)->free == NAPI_GRO_FREE_STOLEN_HEAD)
			kmem_cache_free(skbuff_head_cache, skb);
		else
			__kfree_skb(skb);
		break;

	case GRO_HELD:
	case GRO_MERGED:
		break;
	}

	return ret;
}

int netif_receive_skb(struct sk_buff *skb)
{
	int ret;
    // 设置时间戳
	net_timestamp_check(netdev_tstamp_prequeue, skb);

	if (skb_defer_rx_timestamp(skb))
		return NET_RX_SUCCESS;

	rcu_read_lock();

#ifdef CONFIG_RPS
	if (static_key_false(&rps_needed)) {
		struct rps_dev_flow voidflow, *rflow = &voidflow;
		int cpu = get_rps_cpu(skb->dev, skb, &rflow);

		if (cpu >= 0) {
			ret = enqueue_to_backlog(skb, cpu, &rflow->last_qtail);
			rcu_read_unlock();
			return ret;
		}
	}
#endif
	ret = __netif_receive_skb(skb);
	rcu_read_unlock();
	return ret;
}

static int __netif_receive_skb(struct sk_buff *skb)
{
	int ret;

	if (sk_memalloc_socks() && skb_pfmemalloc(skb)) {
		unsigned long pflags = current->flags;

		/*
		 * PFMEMALLOC skbs are special, they should
		 * - be delivered to SOCK_MEMALLOC sockets only
		 * - stay away from userspace
		 * - have bounded memory usage
		 *
		 * Use PF_MEMALLOC as this saves us from propagating the allocation
		 * context down to all allocation sites.
		 */
		current->flags |= PF_MEMALLOC;
		ret = __netif_receive_skb_core(skb, true);
		tsk_restore_flags(current, pflags, PF_MEMALLOC);
	} else
		ret = __netif_receive_skb_core(skb, false);

	return ret;
}

```

##### 统一处理

中间细节先忽略, 最终调用到 `__netif_receive_skb_core`

```c

static int __netif_receive_skb_core(struct sk_buff *skb, bool pfmemalloc)
{
	struct packet_type *ptype, *pt_prev;
	rx_handler_func_t *rx_handler;
	struct net_device *orig_dev;
	struct net_device *null_or_dev;
	bool deliver_exact = false;
	int ret = NET_RX_DROP;
	__be16 type;

	net_timestamp_check(!netdev_tstamp_prequeue, skb);

	trace_netif_receive_skb(skb);

	/* if we've gotten here through NAPI, check netpoll */
	if (netpoll_receive_skb(skb))
		goto out;

	orig_dev = skb->dev;

	skb_reset_network_header(skb); // 也就是说到这里之后 skb->data 指向了 IP 头
	if (!skb_transport_header_was_set(skb))
		skb_reset_transport_header(skb); // 设置网络层头
	skb_reset_mac_len(skb); // 设置 mac_len

	pt_prev = NULL;

another_round:
	skb->skb_iif = skb->dev->ifindex;

	__this_cpu_inc(softnet_data.processed);

    // 如果带 vlan tag, 先去 tag
	if (skb->protocol == cpu_to_be16(ETH_P_8021Q) ||
	    skb->protocol == cpu_to_be16(ETH_P_8021AD)) {
		skb = vlan_untag(skb);
		if (unlikely(!skb))
			goto out;
	}

#ifdef CONFIG_NET_CLS_ACT
	if (skb->tc_verd & TC_NCLS) {
		skb->tc_verd = CLR_TC_NCLS(skb->tc_verd);
		goto ncls;
	}
#endif

	if (pfmemalloc)
		goto skip_taps;
    // 各种 ptype_all 处理
	list_for_each_entry_rcu(ptype, &ptype_all, list) {
		if (!ptype->dev || ptype->dev == skb->dev) {
			if (pt_prev)
				ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = ptype;
		}
	}

skip_taps:
#ifdef CONFIG_NET_CLS_ACT
	skb = handle_ing(skb, &pt_prev, &ret, orig_dev);
	if (!skb)
		goto out;
ncls:
#endif

	if (pfmemalloc && !skb_pfmemalloc_protocol(skb))
		goto drop;

	if (vlan_tx_tag_present(skb)) {
		if (pt_prev) {
			ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = NULL;
		}
		if (vlan_do_receive(&skb))
			goto another_round;
		else if (unlikely(!skb))
			goto out;
	}
    // 如果注册了 rx_handler, 则直接走设备的 rx_handler, 比如 ovs, bonding
	rx_handler = rcu_dereference(skb->dev->rx_handler);
	if (rx_handler) {
		if (pt_prev) {
			ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = NULL;
		}
		switch (rx_handler(&skb)) {
		case RX_HANDLER_CONSUMED:
			ret = NET_RX_SUCCESS;
			goto out; // 如果设备的 rx_handler 返回RX_HANDLER_CONSUMED, 那么就不走下面的协议栈了
		case RX_HANDLER_ANOTHER:
			goto another_round;
		case RX_HANDLER_EXACT:
			deliver_exact = true;
		case RX_HANDLER_PASS:
			break;
		default:
			BUG();
		}
	}

	if (unlikely(vlan_tx_tag_present(skb))) {
		if (vlan_tx_tag_get_id(skb))
			skb->pkt_type = PACKET_OTHERHOST;
		/* Note: we might in the future use prio bits
		 * and set skb->priority like in vlan_do_receive()
		 * For the time being, just ignore Priority Code Point
		 */
		skb->vlan_tci = 0;
	}

	/* deliver only exact match when indicated */
	null_or_dev = deliver_exact ? skb->dev : NULL;

    // ptype_base 处理, 由各种协议栈处理
	type = skb->protocol; // 比如协议如果是 IP, 那么将从 ip_rcv 开始
	list_for_each_entry_rcu(ptype,
			&ptype_base[ntohs(type) & PTYPE_HASH_MASK], list) {
		if (ptype->type == type &&
		    (ptype->dev == null_or_dev || ptype->dev == skb->dev ||
		     ptype->dev == orig_dev)) {
			if (pt_prev)
				ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = ptype;
		}
	}

	if (pt_prev) {
		if (unlikely(skb_orphan_frags(skb, GFP_ATOMIC)))
			goto drop;
		else
			ret = pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
	} else {
drop:
		atomic_long_inc(&skb->dev->rx_dropped);
		kfree_skb(skb);
		/* Jamal, now you will not able to escape explaining
		 * me how you were going to use this. :-)
		 */
		ret = NET_RX_DROP;
	}

out:
	return ret;
}
```

一般情况下, 系统收到的包都会经 `ptype_base` 进行处理, 各个协议模块使用 `dev_add_pack` 注册, 例如 IP 层的处理函数 `ip_rcv`, TCPv4 的 ` tcp_v4_rcv`   都是是这样挂接到 `ptype_base` 

##### IP 层处理

说下 `ip_rcv`

```c

/*
 * 	Main IP Receive routine.
 */
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt, struct net_device *orig_dev)
{
	const struct iphdr *iph;
	u32 len;

    // 检测各种异常情况, 扔报文
	/* When the interface is in promisc. mode, drop all the crap
	 * that it receives, do not try to analyse it.
	 */
	if (skb->pkt_type == PACKET_OTHERHOST)
		goto drop;


	IP_UPD_PO_STATS_BH(dev_net(dev), IPSTATS_MIB_IN, skb->len);

	if ((skb = skb_share_check(skb, GFP_ATOMIC)) == NULL) {
		IP_INC_STATS_BH(dev_net(dev), IPSTATS_MIB_INDISCARDS);
		goto out;
	}

	if (!pskb_may_pull(skb, sizeof(struct iphdr)))
		goto inhdr_error;

	iph = ip_hdr(skb);

	/*
	 *	RFC1122: 3.2.1.2 MUST silently discard any IP frame that fails the checksum.
	 *
	 *	Is the datagram acceptable?
	 *
	 *	1.	Length at least the size of an ip header
	 *	2.	Version of 4
	 *	3.	Checksums correctly. [Speed optimisation for later, skip loopback checksums]
	 *	4.	Doesn't have a bogus length
	 */

	if (iph->ihl < 5 || iph->version != 4)
		goto inhdr_error;

	if (!pskb_may_pull(skb, iph->ihl*4))
		goto inhdr_error;

	iph = ip_hdr(skb);

	if (unlikely(ip_fast_csum((u8 *)iph, iph->ihl)))
		goto csum_error;

	len = ntohs(iph->tot_len);
	if (skb->len < len) {
		IP_INC_STATS_BH(dev_net(dev), IPSTATS_MIB_INTRUNCATEDPKTS);
		goto drop;
	} else if (len < (iph->ihl*4))
		goto inhdr_error;

	/* Our transport medium may have padded the buffer out. Now we know it
	 * is IP we can trim to the true length of the frame.
	 * Note this now means skb->len holds ntohs(iph->tot_len).
	 */
	if (pskb_trim_rcsum(skb, len)) {
		IP_INC_STATS_BH(dev_net(dev), IPSTATS_MIB_INDISCARDS);
		goto drop;
	}
    // 设置传输层的头, 供后面协议使用
	skb->transport_header = skb->network_header + iph->ihl*4;

	/* Remove any debris in the socket control block */
	memset(IPCB(skb), 0, sizeof(struct inet_skb_parm));

	/* Must drop socket now because of tproxy. */
	skb_orphan(skb);
    // netfilter 框架的 ipv4 pre routing 入口, 比如 iptables 的部分规则就靠这个了
	return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, skb, dev, NULL,
		       ip_rcv_finish);

csum_error:
	IP_INC_STATS_BH(dev_net(dev), IPSTATS_MIB_CSUMERRORS);
inhdr_error:
	IP_INC_STATS_BH(dev_net(dev), IPSTATS_MIB_INHDRERRORS);
drop:
	kfree_skb(skb);
out:
	return NET_RX_DROP;
}

```

上面代码中最重要的一行就是 转由 netfilter 框架处理的部分, 经过 netfilter 之后, 如果需要后续处理, netfilter 返回 1, 然后调用 `ip_rcv_finish`

`__netif_receive_skb_core` 的说明中提到了 `dev->rx_handler` , 比如 ovs, 这里就有一个关键点了, 就是一个网络设备被 ovs 注册成了 bridge 之后, 这个网络设备收到的包会走到 ovs 的内部处理流程, 后面的协议栈就不走了, 那么 `iptables` 对这个网络设备就失效了

openstack 里使用 ovs 时, 下发一些安全规则的时候需要 iptables, 怎么实现的? 建立一个 Linux bridge, 用 vpair 连上 ovs 和 Linux bridge, 在这个  Linux bridge 上下发规则就可以了. 如下图 `br-int` 和 租户中间的网络结构 ([来源](https://docs.openstack.org/developer/neutron/devref/openvswitch_agent.html))

![](https://docs.openstack.org/developer/neutron/_images/under-the-hood-scenario-1-ovs-compute.png)



接下来 `ip_rcv_finish`

```c
static int ip_rcv_finish(struct sk_buff *skb)
{
	const struct iphdr *iph = ip_hdr(skb);
	struct rtable *rt;

	if (sysctl_ip_early_demux && !skb_dst(skb) && skb->sk == NULL) {
		const struct net_protocol *ipprot;
		int protocol = iph->protocol;

		ipprot = rcu_dereference(inet_protos[protocol]);
		if (ipprot && ipprot->early_demux) {
			ipprot->early_demux(skb);
			/* must reload iph, skb->head might have changed */
			iph = ip_hdr(skb);
		}
	}

	/*
	 *	Initialise the virtual path cache for the packet. It describes
	 *	how the packet travels inside Linux networking.
	 */
    // 这次的分析从网卡收上来的报文流程, 没有路由信息, 所以会进到这个分支
	if (!skb_dst(skb)) {
		int err = ip_route_input_noref(skb, iph->daddr, iph->saddr,
					       iph->tos, skb->dev); // 查路由表
		if (unlikely(err)) {
			if (err == -EXDEV)
				NET_INC_STATS_BH(dev_net(skb->dev),
						 LINUX_MIB_IPRPFILTER);
			goto drop;
		}
	}

#ifdef CONFIG_IP_ROUTE_CLASSID
	if (unlikely(skb_dst(skb)->tclassid)) {
		struct ip_rt_acct *st = this_cpu_ptr(ip_rt_acct);
		u32 idx = skb_dst(skb)->tclassid;
		st[idx&0xFF].o_packets++;
		st[idx&0xFF].o_bytes += skb->len;
		st[(idx>>16)&0xFF].i_packets++;
		st[(idx>>16)&0xFF].i_bytes += skb->len;
	}
#endif
    // 如果头部有 option, 先处理 option
	if (iph->ihl > 5 && ip_rcv_options(skb))
		goto drop;

	rt = skb_rtable(skb); // 取路由表
	if (rt->rt_type == RTN_MULTICAST) {
		IP_UPD_PO_STATS_BH(dev_net(rt->dst.dev), IPSTATS_MIB_INMCAST,
				skb->len);
	} else if (rt->rt_type == RTN_BROADCAST)
		IP_UPD_PO_STATS_BH(dev_net(rt->dst.dev), IPSTATS_MIB_INBCAST,
				skb->len);
    // 根据路由信息调用 ip_local_deliver(本地处理) 或 ip_forward(转发) 或 ip_error(icmp返回错误信息)
	return dst_input(skb);

drop:
	kfree_skb(skb);
	return NET_RX_DROP;
}
```

```c
/*
 * 	Deliver IP Packets to the higher protocol layers.
 */
int ip_local_deliver(struct sk_buff *skb)
{
	/*
	 *	Reassemble IP fragments.  将分片报文合并
	 */
	if (ip_is_fragment(ip_hdr(skb))) {
		if (ip_defrag(skb, IP_DEFRAG_LOCAL_DELIVER))
			return 0;
	}
    // 转交 netfilter 框架
	return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN, skb, skb->dev, NULL,
		       ip_local_deliver_finish);
}

static int ip_local_deliver_finish(struct sk_buff *skb)
{
	struct net *net = dev_net(skb->dev);

	__skb_pull(skb, skb_network_header_len(skb));

	rcu_read_lock();
	{
		int protocol = ip_hdr(skb)->protocol;
		const struct net_protocol *ipprot;
		int raw;

	resubmit:
		raw = raw_local_deliver(skb, protocol);

		ipprot = rcu_dereference(inet_protos[protocol]);
		if (ipprot != NULL) {
			int ret;

			if (!ipprot->no_policy) {
				if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
					kfree_skb(skb);
					goto out;
				}
				nf_reset(skb);
			}
			ret = ipprot->handler(skb); // 调上层协议模块处理
			if (ret < 0) {
				protocol = -ret;
				goto resubmit;
			}
			IP_INC_STATS_BH(net, IPSTATS_MIB_INDELIVERS);
		} else {
			if (!raw) {
				if (xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
					IP_INC_STATS_BH(net, IPSTATS_MIB_INUNKNOWNPROTOS);
					icmp_send(skb, ICMP_DEST_UNREACH,
						  ICMP_PROT_UNREACH, 0);
				}
				kfree_skb(skb);
			} else {
				IP_INC_STATS_BH(net, IPSTATS_MIB_INDELIVERS);
				consume_skb(skb);
			}
		}
	}
 out:
	rcu_read_unlock();

	return 0;
}
```

##### 更上层处理

IP 上层的协议比较分散 ( ICMP/IGMP/TCP/UDP 等), 更接近应用层, 侧重点也不一样, 这里不再进一步走了, 上层协议使用这个 skb 之后可能需要发包, 比如 TCP 模块调用 `ip_local_out` 来发包



未完待续呀..



**参考**

* [http://vger.kernel.org/~davem/skb.html](http://vger.kernel.org/~davem/skb.html)
* [UDP Encapsulation in Linux](https://www.netdev01.org/docs/herbert-UDP-Encapsulation-Linux.pdf)
* [Packet fragmentation and segmentation offload in UDP and VXLAN](http://hustcat.github.io/udp-and-vxlan-fragment/)
* [Linux 下网络性能优化方法简析](https://www.ibm.com/developerworks/cn/linux/l-cn-network-pt/)