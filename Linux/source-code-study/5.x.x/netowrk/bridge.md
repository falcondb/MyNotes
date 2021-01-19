## Bridge

### Key data structures
* `br_private.h`
```
struct net_bridge {
	spinlock_t			lock;
	spinlock_t			hash_lock;
	struct list_head		port_list;		// list of net_bridge_port
	struct net_device		*dev;					// the bridge net_device
	struct pcpu_sw_netstats		__percpu *stats;
	unsigned long			options;
	__be16				    vlan_proto;
	u16				        default_pvid;
	struct net_bridge_vlan_group	__rcu *vlgrp;
	struct rhashtable		fdb_hash_tbl;		// forwarding DB
	union {
		struct rtable		  fake_rtable;
		struct rt6_info		fake_rt6_info;
	};
	u16				group_fwd_mask;
	u16				group_fwd_mask_required;
	/* STP */
	bridge_id			designated_root;
	bridge_id			bridge_id;
	unsigned char			topology_change;
	unsigned char			topology_change_detected;
	u16				        root_port;
	unsigned long			max_age;
	unsigned long			hello_time;
	unsigned long			forward_delay;
	unsigned long			ageing_time;
	unsigned long			bridge_max_age;
	unsigned long			bridge_hello_time;
	unsigned long			bridge_forward_delay;
	unsigned long			bridge_ageing_time;
	u32				root_path_cost;
	u8				group_addr[ETH_ALEN];
	enum {
		BR_NO_STP, 		/* no spanning tree */
		BR_KERNEL_STP,		/* old STP in kernel */
		BR_USER_STP,		/* new RSTP in userspace */
	} stp_enabled;
	u32				hash_max;
	u32				multicast_last_member_count;
	u32				multicast_startup_query_count;
	u8				multicast_igmp_version;
	u8				multicast_router;
	u8				multicast_mld_version;
	spinlock_t			multicast_lock;
	unsigned long			multicast_last_member_interval;
	unsigned long			multicast_membership_interval;
	unsigned long			multicast_querier_interval;
	unsigned long			multicast_query_interval;
	unsigned long			multicast_query_response_interval;
	unsigned long			multicast_startup_query_interval;
	struct rhashtable		mdb_hash_tbl;
	struct hlist_head		mdb_list;
	struct hlist_head		router_list;
	struct timer_list		multicast_router_timer;
	struct bridge_mcast_other_query	ip4_other_query;
	struct bridge_mcast_own_query	ip4_own_query;
	struct bridge_mcast_querier	ip4_querier;
	struct bridge_mcast_stats	__percpu *mcast_stats;
	struct bridge_mcast_other_query	ip6_other_query;
	struct bridge_mcast_own_query	ip6_own_query;
	struct bridge_mcast_querier	ip6_querier;
	struct timer_list		hello_timer;
	struct timer_list		tcn_timer;
	struct timer_list		topology_change_timer;
	struct delayed_work		gc_work;
	struct kobject			*ifobj;
	u32				auto_cnt;
	int offload_fwd_mark;
	struct hlist_head		fdb_list;
};


struct net_bridge_port {
	struct net_bridge		*br;
	struct net_device		*dev;
	struct list_head		list;
	unsigned long			flags;
	struct net_bridge_vlan_group	__rcu *vlgrp;
	struct net_bridge_port		__rcu *backup_port;

	/* STP */
	u8				priority;
	u8				state;
	u16				port_no;
	unsigned char			topology_change_ack;
	unsigned char			config_pending;
	port_id				port_id;
	port_id				designated_port;
	bridge_id			designated_root;
	bridge_id			designated_bridge;
	u32				path_cost;
	u32				designated_cost;
	unsigned long			designated_age;
	struct timer_list		forward_delay_timer;
	struct timer_list		hold_timer;
	struct timer_list		message_age_timer;
	struct kobject			kobj;
	struct rcu_head			rcu;
	struct bridge_mcast_own_query	ip4_own_query;
	struct bridge_mcast_own_query	ip6_own_query;
	unsigned char			multicast_router;
	struct bridge_mcast_stats	__percpu *mcast_stats;
	struct timer_list		multicast_router_timer;
	struct hlist_head		mglist;
	struct hlist_node		rlist;
	char				sysfs_name[IFNAMSIZ];
	struct netpoll			*np;
	u16				group_fwd_mask;
	u16				backup_redirected_cnt;
};


struct net_bridge_fdb_entry {
	struct rhash_head		rhnode;
	struct net_bridge_port		*dst;		// MAC addr last seen
	struct net_bridge_fdb_key	key;		// MAC + vlan
	struct hlist_node		fdb_node;
	unsigned long			flags;
	unsigned long			updated ____cacheline_aligned_in_smp;
	unsigned long			used;
	struct rcu_head			rcu;
};


struct net_bridge_fdb_key {
	mac_addr addr;
	u16 vlan_id;
};


struct net_bridge_vlan {
	struct rhash_head		vnode;
	struct rhash_head		tnode;
	u16				vid;
	u16				flags;
	u16				priv_flags;
	struct br_vlan_stats __percpu	*stats;
	union {
		struct net_bridge	*br;
		struct net_bridge_port	*port;
	};
	union {
		refcount_t		refcnt;
		struct net_bridge_vlan	*brvlan;
	};

	struct br_tunnel_info		tinfo;

	struct list_head		vlist;

	struct rcu_head			rcu;
};
```

### Initialization
* `br.c`
```
__init br_init
```

* `br_device.c`
```
br_dev_setup
```

### Key functions

* `br_ioctl.c`
```
br_dev_ioctl
```

* `br_input.c`
```
br_handle_frame
	p = br_port_get_rcu(skb->dev)
	if BR_VLAN_TUNNEL
			br_handle_ingress_vlan_tunnel(skb, p, nbp_vlan_group_rcu(p))

	// link local reserved addr (01:80:c2:00:00:0X) per IEEE 802.1Q 8.6.3 Frame filtering
	if is_link_local_ether_addr
		call __br_handle_local_finish or continue in br_handle_frame

	switch net_bridge_port p->state
		BR_STATE_FORWARDING:
			rhook = rcu_dereference(br_should_route_hook)
			(*rhook)(skb)
				// return !0 continue in bridge, otherwise jump to network layer
		BR_STATE_LEARNING:
			NF_HOOK(NFPROTO_BRIDGE, NF_BR_PRE_ROUTING,br_handle_frame_finish)

```

br_handle_frame_finish called after bridge netfilter hook, NF_BR_PRE_ROUTING
```
br_handle_frame_finish
	br_fdb_update		// updates the fdb with the source MAC, and the source interface

	BR_INPUT_SKB_CB(skb)->brdev = br->dev

	IPv4 ARP/RARP: br_do_proxy_suppress_arp
	IPv6 ND: br_do_suppress_nd

	switch pkt_type
		BR_PKT_MULTICAST:
			mdst = br_mdb_get
		BR_PKT_UNICAST:
			dst = br_fdb_find_rcu		==>		fdb_find_rcu	// find a forwarding port for unicast

	if dst
		if dst->is_local
			return br_pass_frame_up
		br_forward
	else				// forwarding port for unicast is not found, flood to all ports
		br_flood or br_multicast_flood

	if local_rcv
		br_pass_frame_up
```

```
br_pass_frame_up
  br_handle_vlan
	NF_HOOK(NFPROTO_BRIDGE, NF_BR_LOCAL_IN, br_netif_receive_skb)
	// br_netif_receive_skb		==>	 netif_receive_skb
```

* `br_forward.c`

```
br_forward
	/* redirect to backup link if the destination port is down */
	if should_deliver	// /* Don't forward packets to originating port or forwarding disabled */
		if local_rcv
			deliver_clone
		else
			__br_forward
```

```
br_flood
	for each port in br->port_list
		// don't forward to ports with mismatched configuration
		maybe_deliver

	if local_rcv
		deliver_clone
	else
		__br_forward
```

```
__br_forward
	br_handle_vlan

	if !local_orig
		br_hook = NF_BR_FORWARD
	else
		br_hook = NF_BR_LOCAL_OUT

		NF_HOOK(NFPROTO_BRIDGE, br_hook, ..., br_forward_finish);
```

```
br_forward_finish
NF_HOOK(NFPROTO_BRIDGE, NF_BR_POST_ROUTING, ..., br_dev_queue_push_xmit)
	// br_dev_queue_push_xmit: if vlan, skb_set_network_header. dev_queue_xmit
```

```
deliver_clone
	skb_clone
	__br_forward
```

```
maybe_deliver
	if !should_deliver
		return

	deliver_clone
```

#### Bridge VLAN
* `br_vlan.c`
```
br_handle_vlan
	// if should use vlan, kfree_skb and return null
	br_handle_egress_vlan_tunnel
		br_vlan_tunnel_lookup
		skb_dst_drop
		__vlan_hwaccel_put_tag(skb, p->br->vlan_proto, vlan->vid)
```

#### Bridge routing
##### Initialization

* `ebtable_broute.c`
```
__init ebtable_broute_init
	register_pernet_subsys(&broute_net_ops)

	RCU_INIT_POINTER(br_should_route_hook, ebt_broute)

```

```
ebt_broute
	nf_hook_state_init
	ebt_do_table(skb, &state, state.net->xt.broute_table)
```

* `ebtables.c`
```
ebt_do_table


```
