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
