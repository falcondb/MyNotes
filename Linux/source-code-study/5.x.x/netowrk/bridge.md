## Bridge

### Key data structures
* `br_private.h`
```
struct net_bridge {
	spinlock_t			lock;
	spinlock_t			hash_lock;
	struct list_head		port_list;
	struct net_device		*dev;
	struct pcpu_sw_netstats		__percpu *stats;
	unsigned long			options;
	__be16				    vlan_proto;
	u16				        default_pvid;
	struct net_bridge_vlan_group	__rcu *vlgrp;
	struct rhashtable		fdb_hash_tbl;
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
	struct net_bridge_port		*dst;
	struct net_bridge_fdb_key	key;
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

br_pass_frame_up
  br_handle_vlan
  NF_HOOK
```

* `br_forward.c`

```
br_forward

br_flood
```

* `br_vlan.c`
