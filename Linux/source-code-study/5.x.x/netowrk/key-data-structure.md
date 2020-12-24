# Key data structures and function prototypes

### include/linux/skbuff.h ###
* struct sk_buff {}

### net/netfilter/nf_tables_core.h/c ###
* nf table operations

### net/core/dev.c ###
* network device operations

### include/linux/skmsg.h
* sk_psock_progs
```
struct sk_psock_progs {
	struct bpf_prog			*msg_parser;
	struct bpf_prog			*skb_parser;
	struct bpf_prog			*skb_verdict;
};
```

#### net/net_namespace.h
```
struct net {
	refcount_t		passive;	/* To decide when should be freed */
	refcount_t		count;		/* To decided when should be shut down */
	spinlock_t		rules_mod_lock;
	unsigned int		dev_unreg_count;
	unsigned int		dev_base_seq;	/* protected by rtnl_mutex */
	int			ifindex;

	spinlock_t		nsid_lock;
	atomic_t		fnhe_genid;

	struct list_head	list;		/* list of network namespaces */
	struct list_head	exit_list;
	struct llist_node	cleanup_list;	/* namespaces on death row */

	struct user_namespace   *user_ns;	/* Owning user namespace */
	struct ucounts		*ucounts;
	struct idr		netns_ids;

	struct ns_common	ns;

	struct list_head 	dev_base_head;
	struct proc_dir_entry 	*proc_net;
	struct proc_dir_entry 	*proc_net_stat;

	struct ctl_table_set	sysctls;
	struct sock		*genl_sock;
	struct uevent_sock	*uevent_sock;		/* uevent socket */
	struct hlist_head 	*dev_name_head;
	struct hlist_head	*dev_index_head;
	struct raw_notifier_head	netdev_chain;
	u32			hash_mix;
	struct net_device       *loopback_dev;          /* The loopback */
	struct list_head	rules_ops;
	struct netns_core	core;
	struct netns_mib	mib;
	struct netns_packet	packet;
	struct netns_unix	unx;
	struct netns_nexthop	nexthop;
	struct netns_ipv4	ipv4;
	struct netns_ipv6	ipv6;
	struct netns_sctp	sctp;
	struct netns_dccp	dccp;
	struct netns_nf		nf;
	struct netns_xt		xt;
	struct netns_ct		ct;
	struct netns_nftables	nft;
	struct netns_nf_frag	nf_frag;
	struct ctl_table_header *nf_frag_frags_hdr;
	struct sock		*nfnl;
	struct sock		*nfnl_stash;
	struct netns_xfrm	xfrm;
	...
	struct netns_xdp	xdp;
	...
} __randomize_layout;


struct pernet_operations {
	struct list_head list;
	int (*init)(struct net *net);
	void (*pre_exit)(struct net *net);
	void (*exit)(struct net *net);
	void (*exit_batch)(struct list_head *net_exit_list);
	unsigned int *id;
	size_t size;
};

```


#### linux/netdevice.h
```
struct net_device {
	char			name[IFNAMSIZ];
	struct netdev_name_node	*name_node;
	struct dev_ifalias	__rcu *ifalias;
	unsigned long		mem_end;
	unsigned long		mem_start;
	unsigned long		base_addr;			// I/O basic address
	int			irq;
	unsigned long		state;
	struct list_head	dev_list napi_list unreg_list close_list ptype_all ptype_specific;
	struct {
		struct list_head upper;
		struct list_head lower;
	} adj_list;
	netdev_features_t	features hw_features wanted_features vlan_features hw_enc_features mpls_features gso_partial_features;
	int			ifindex;
	int			group;
	struct net_device_stats	stats;
	atomic_long_t		rx_dropped tx_dropped rx_nohandler;
	atomic_t		carrier_up_count carrier_down_count;
	const struct net_device_ops *netdev_ops;
	const struct ethtool_ops *ethtool_ops;
	const struct ndisc_ops *ndisc_ops;
	const struct xfrmdev_ops *xfrmdev_ops;
	const struct tlsdev_ops *tlsdev_ops;
	const struct header_ops *header_ops;

	unsigned int		flags priv_flags;
	unsigned short		gflags;
	unsigned short		padded;

	unsigned char		operstate;
	unsigned char		link_mode;
	unsigned char		if_port;
	unsigned char		dma;
	unsigned int		mtu min_mtu max_mtu;
	unsigned short	type;
	unsigned short	hard_header_len;
	unsigned char		min_header_len;
	unsigned short	needed_headroom needed_tailroom;

	unsigned char		perm_addr[MAX_ADDR_LEN];
	unsigned char		addr_assign_type;
	unsigned char		addr_len;
	unsigned char		upper_level lower_level;
	unsigned short	eigh_priv_len;
	unsigned short  dev_id;
	unsigned short  dev_port;
	spinlock_t			addr_list_lock;
	unsigned char		name_assign_type;
	bool						uc_promisc;
	struct netdev_hw_addr_list	uc mc dev_addrs;
	struct kset			*queues_kset;

	unsigned int		promiscuity allmulti;

	struct vlan_info __rcu	*vlan_info;
	struct dsa_port		*dsa_ptr;
	...

	unsigned char		*dev_addr;
	struct netdev_rx_queue	*_rx;
	unsigned int		num_rx_queues;
	unsigned int		real_num_rx_queues;

	struct bpf_prog __rcu	*xdp_prog;
	unsigned long		gro_flush_timeout;
	rx_handler_func_t __rcu	*rx_handler;
	void __rcu		*rx_handler_data;
	struct mini_Qdisc __rcu	*miniq_ingress;
	struct netdev_queue __rcu *ingress_queue;
	struct nf_hook_entries __rcu *nf_hooks_ingress;
	unsigned char		broadcast[MAX_ADDR_LEN];
	struct hlist_node	index_hlist;

	struct netdev_queue	*_tx ____cacheline_aligned_in_smp;
	unsigned int		num_tx_queues;
	unsigned int		real_num_tx_queues;
	struct Qdisc		*qdisc;
	DECLARE_HASHTABLE	(qdisc_hash, 4);
	unsigned int		tx_queue_len;
	spinlock_t		tx_global_lock;
	int			watchdog_timeo;
	struct mini_Qdisc __rcu	*miniq_egress;
	struct timer_list	watchdog_timer;

	int __percpu		*pcpu_refcnt;
	struct list_head	todo_list;

	struct list_head	link_watch_list;
	reg_state:8;
	bool dismantle;
	rtnl_link_state:16;
	bool needs_free_netdev;
	void (*priv_destructor)(struct net_device *dev);
	struct netpoll_info __rcu	*npinfo;
	possible_net_t			nd_net;

	union {
		void					*ml_priv;
		struct pcpu_lstats __percpu		*lstats;
		struct pcpu_sw_netstats __percpu	*tstats;
		struct pcpu_dstats __percpu		*dstats;
	};

	struct device		dev;
	const struct attribute_group *sysfs_groups[4];
	const struct attribute_group *sysfs_rx_queue_group;
	const struct rtnl_link_ops *rtnl_link_ops;

	s16			num_tc;
	struct netdev_tc_txq	tc_to_txq[TC_MAX_QUEUE];
	u8			prio_tc_map[TC_BITMASK + 1];
	struct netprio_map __rcu *priomap;
	struct phy_device	*phydev;
	struct sfp_bus		*sfp_bus;
	struct lock_class_key	qdisc_tx_busylock_key;
	struct lock_class_key	qdisc_running_key;
	struct lock_class_key	qdisc_xmit_lock_key;
	struct lock_class_key	addr_list_lock_key;
	bool			proto_down;
	unsigned		wol_enabled:1;
};
```
