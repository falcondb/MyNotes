## IP layer

### net
* `netif_rx`

### IPv4
See `struct packet_type` definition in dev.md
```
static struct packet_type ip_packet_type __read_mostly = {
	.type = cpu_to_be16(ETH_P_IP),
	.func = ip_rcv,
	.list_func = ip_list_rcv,
};
```

### key functions

#### IPv4 initialization
* `net/ipv4/af_inet.c`
```
__init inet_init

```


#### ingress
##### `ip_input.c`
* `ip_rcv`

`ip_rcv` is registered as the callback in `packet_type` for IP protocol. `dev` calls it when the packet is ready to be processed in IP layer.
```
ip_rcv
	ip_rcv_core
		/ sanity check of the IP header and the skb
	NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, ..., ip_rcv_finsih)
```


* `ip_rcv_finsih`
```
ip_rcv_finsih
	ip_rcv_finish_core
		// see below
	dst_input
		skb_dst(skb)->input
```

Attempts to call the early_demux function from the higher level protocol that this data is destined for. The early_demux routine is an optimization which attempts to find the dst_entry needed to deliver the packet by checking if a dst_entry is cached on the socket structure.
```
ip_rcv_finish_core
	// early_demux optimization
	if net->ipv4.sysctl_ip_early_demux
		edemux = ipprot->early_demux
		edemux(skb)

	// new packet or !early_demux
	ip_rcv_options
	if !skb_valid_dst
		ip_route_input_noref	// ip_forward
	rt = skb_rtable(skb)
```

* `ip_local_deliver`
Set `ip_local_deliver` to `rttable.dst.input = ip_local_deliver` in `rt_dst_alloc() ipv4/route.c` for local delivery
```
ip_local_deliver
	if ip_is_fragment ==> ip_defrag; return 0
	NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN, ip_local_deliver_finish)
```

* `ip_defrag` in `ipv4/ip_fragment.c`
```
ip_defrag

```

* `ip_local_deliver_finish`
```
ip_local_deliver_finish
	ip_protocol_deliver_rcu(net, skb, ip_hdr(skb)->protocol)

	ipprot = rcu_dereference(inet_protos[protocol])
	ipprot->handler(skb)	// to upper layer

	// some xfrm4_policy_check here for IPSec
```


#### ip_forward
* `ip_forward`

* `ip_forward_finish`

* ` ip_route_input` in `net/route.h`

* `ip_queue_xmit`

#### egress

* `ip_send_skb` & `ip_local_out`
```
ip_send_skb	==> ip_local_out

```
```
ip_local_out
	__ip_local_out
		l3mdev_ip_out // see l3mdev.h and paper "what What is an L3 Master Device?"
		nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT, dst_output)
	dst_output
		(skb->_skb_refdst & SKB_DST_PTRMASK)->output()
```

* `__ip_queue_xmit` in `ipv4/ip_output.c`
```
ip_queue_xmit  ==>  __ip_queue_xmit
```

* `p_queue_xmit2`?

* `ip_output`
```
NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, ip_finish_output, !(IPCB(skb)->flags & IPSKB_REROUTED))
```

* `ip_finish_output`
```
ip_finish_output
	if skb_is_gso
		ip_finish_output_gso

	if skb->len > mtu
		ip_fragment

	ip_finish_output2

```

```
ip_finish_output2
	nexthop = rt_nexthop(rt, ip_hdr(skb)->daddr)
		rt->rt_gw_family == AF_INET? rt->rt_gw4: daddr
	neigh = __ipv4_neigh_lookup_noref(dev, nexthop)	\\ see neighbour.md
	sock_confirm_neigh(skb, neigh)
	neigh_output(neigh, skb)		\\ see neighbour.md
		neigh_hh_output or struct neighbour->output
```

* `ip_fragment`

* `dev_queue_xmit`




#### IP options
##### Key data structures
* `inet_sock.h`
```
/** struct ip_options - IP Options
 *
 * @faddr - Saved first hop address
 * @nexthop - Saved nexthop address in LSRR and SSRR (Loose Source Routing, Strict Source Routing)
 * @is_strictroute - Strict source route
 * @srr_is_hit - Packet destination addr was our one
 * @is_changed - IP checksum more not valid
 * @rr_needaddr - Need to record addr of outgoing dev
 * @ts_needtime - Need to record timestamp
 * @ts_needaddr - Need to record addr of outgoing dev
 */
struct ip_options {
	__be32		faddr;
	__be32		nexthop;
	unsigned char	optlen;
	unsigned char	srr;   //
	unsigned char	rr;    //Record Route
	unsigned char	ts;    // timestamp
	unsigned char	is_strictroute:1,
			srr_is_hit:1,
			is_changed:1,
			rr_needaddr:1,
			ts_needtime:1,
			ts_needaddr:1;
	unsigned char	router_alert;
	unsigned char	cipso;
	unsigned char	__pad2;
	unsigned char	__data[0];
};
```

##### Key functions
* `net/ipv4/ip_options.c`
```
ip_options_compile    // ingress

ip_options_build      // egress

__ip_options_echo

ip_options_fragment

```

#### ICMP

##### Initialization
```
static struct pernet_operations __net_initdata icmp_sk_ops = {
       .init = icmp_sk_init,
       .exit = icmp_sk_exit,
};
__init icmp_init
  register_pernet_subsys(&icmp_sk_ops)
```

##### Key functions
```
icmp_send
```

```
icmp_rcv

```

```
icmp_reply
```

```
icmp_redirect
```

```
icmp_unreach
```

```
icmp_echo
```

```
icmp_timestamp
```


#### IGMP
##### Key data structures
* `igmp.h`
```
struct ip_mc_list {
	struct in_device	*interface;
	__be32			multiaddr;
	unsigned int		sfmode;
	struct ip_sf_list	*sources;
	struct ip_sf_list	*tomb;
	unsigned long		sfcount[2];
	union {
		struct ip_mc_list *next;
		struct ip_mc_list __rcu *next_rcu;
	};
	struct ip_mc_list __rcu *next_hash;
	struct timer_list	timer;
	int			users;
	refcount_t		refcnt;
	spinlock_t		lock;
	char			tm_running;
	char			reporter;
	char			unsolicit_count;
	char			loaded;
	unsigned char		gsquery;	/* check source marks? */
	unsigned char		crcount;
	struct rcu_head		rcu;
};
```

* `mroute_base.h`
```
struct vif_device {
	struct net_device *dev;
	unsigned long bytes_in, bytes_out;
	unsigned long pkt_in, pkt_out;
	unsigned long rate_limit;
	unsigned char threshold;   // TTL threshold
	unsigned short flags;      // a tunnel VIFF_TUNNEL?
	int link;                  // index of physical device

	/* Currently only used by ipmr */
	struct netdev_phys_item_id dev_parent_id;
	__be32 local, remote;
};

/**
 * struct mr_table - a multicast routing table
 * @list: entry within a list of multicast routing tables
 * @net: net where this table belongs
 * @ops: protocol specific operations
 * @id: identifier of the table
 * @mroute_sk: socket associated with the table
 * @ipmr_expire_timer: timer for handling unresolved routes
 * @mfc_unres_queue: list of unresolved MFC entries
 * @vif_table: array containing all possible vifs
 * @mfc_hash: Hash table of all resolved routes for easy lookup
 * @mfc_cache_list: list of resovled routes for possible traversal
 * @maxvif: Identifier of highest value vif currently in use
 * @cache_resolve_queue_len: current size of unresolved queue
 * @mroute_do_assert: Whether to inform userspace on wrong ingress
 * @mroute_do_pim: Whether to receive IGMP PIMv1
 * @mroute_reg_vif_num: PIM-device vif index
 */
struct mr_table {
	struct list_head	list;
	possible_net_t		net;
	struct mr_table_ops	ops;
	u32			id;
	struct sock __rcu	*mroute_sk;
	struct timer_list	ipmr_expire_timer;
	struct list_head	mfc_unres_queue;
	struct vif_device	vif_table[MAXVIFS];
	struct rhltable		mfc_hash;
	struct list_head	mfc_cache_list;
	int			maxvif;
	atomic_t		cache_resolve_queue_len;
	bool			mroute_do_assert;
	bool			mroute_do_pim;
	bool			mroute_do_wrvifwhole;
	int			mroute_reg_vif_num;
};
```

* `mroute.h`
```
/**
 * struct mr_mfc - common multicast routing entries
 * @mnode: rhashtable list
 * @mfc_parent: source interface (iif)
 * @mfc_flags: entry flags
 * @expires: unresolved entry expire time
 * @unresolved: unresolved cached skbs
 * @last_assert: time of last assert
 * @minvif: minimum VIF id
 * @maxvif: maximum VIF id
 * @bytes: bytes that have passed for this entry
 * @pkt: packets that have passed for this entry
 * @wrong_if: number of wrong source interface hits
 * @lastuse: time of last use of the group (traffic or update)
 * @ttls: OIF TTL threshold array
 * @refcount: reference count for this entry
 * @list: global entry list
 * @rcu: used for entry destruction
 * @free: Operation used for freeing an entry under RCU
 */
struct mr_mfc {
	struct rhlist_head mnode;  
	unsigned short mfc_parent;     // the index of the virtual network device in the vif_table
	int mfc_flags;
	union {
		struct {
			unsigned long expires;
			struct sk_buff_head unresolved;
		} unres;
		struct {
			unsigned long last_assert;
			int minvif;
			int maxvif;
			unsigned long bytes;
			unsigned long pkt;
			unsigned long wrong_if;
			unsigned long lastuse;
			unsigned char ttls[MAXVIFS];
			refcount_t refcount;
		} res;
	} mfc_un;
	struct list_head list;
	struct rcu_head	rcu;
	void (*free)(struct rcu_head *head);
};

/**
 * struct mfc_cache - multicast routing entries
 * @_c: Common multicast routing information; has to be first [for casting]
 * @mfc_mcastgrp: destination multicast group address
 * @mfc_origin: source address
 * @cmparg: used for rhashtable comparisons
 */
struct mfc_cache {
	struct mr_mfc _c;
	union {
		struct {
			__be32 mfc_mcastgrp;
			__be32 mfc_origin;
		};
		struct mfc_cache_cmp_arg cmparg;
	};
};
```

##### Initialization
```
__init igmp_mc_init
```

##### Key functions

###### Control plane
* `net/ipv4/igmp.c`
```
igmp_rcv

igmp_heard_report

igmp_heard_query

igmp_send_report
```

* `net/ipv4/ipmr.c`     // mrounted support
```
__init ip_mr_init

ip_mroute_setsockopt    // mrouted interface

ipmr_ioctl              // ioctl interface

ipmr_get_route

ipmr_cache_find

ipmr_new_tunnel
```

###### Data plane
* `ipv4/route.c`
```
ip_route_input_mc
  ip_mc_validate_source
  flags |= RTCF_LOCAL
  struct rtable rth = rt_dst_alloc
  rth->dst.input = ip_mr_input
```

* `ipv4/ipmr.c`

```
ip_mr_input
```

```
ip_mr_forward
  dst->output
```

```
ipmr_queue_xmit
```
