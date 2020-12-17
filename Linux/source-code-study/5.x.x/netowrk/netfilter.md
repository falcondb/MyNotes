## NetFilter

### Key data structures

* `nf_flow_table.h`

```
struct nf_flowtable_type {
	struct list_head		list;
	int				family;
	int				(*init)   (struct nf_flowtable *);
	int				(*setup)  (struct nf_flowtable *, struct net_device *, enum flow_block_command);
	int				(*action) (struct net, struct flow_offload, enum flow_offload_tuple_dir, struct nf_flow_rule);
	void			(*free)   (struct nf_flowtable *);
	nf_hookfn			     *hook;
	struct module			 *owner;
};
```

```
struct nf_flowtable {
	struct list_head		list;
	struct rhashtable		rhashtable;
	int				priority;
	struct nf_flowtable_type	*type;
	struct delayed_work		gc_work;
	unsigned int			flags;
	struct flow_block		flow_block;
	possible_net_t			net;
};
```

* `flow_offload.h`

```
struct flow_action {
	unsigned int			num_entries;
	struct flow_action_entry 	entries[0];
};
```

```
struct flow_action_entry {
	enum flow_action_id		id;         /* FLOW_ACTION_XXXX */
	action_destr			destructor;
	void				*destructor_priv;
	union {
		u32			chain_index;	          /* FLOW_ACTION_GOTO */
		struct net_device	*dev;		      /* FLOW_ACTION_REDIRECT */
		struct {				                /* FLOW_ACTION_VLAN */
			u16		vid;
			__be16		proto;
			u8		prio;
		} vlan;
		struct {				                /* FLOW_ACTION_PACKET_EDIT */
			enum flow_action_mangle_base htype;
			u32		offset;
			u32		mask;
			u32		val;
		} mangle;
		struct ip_tunnel_info	*tunnel;	/* FLOW_ACTION_TUNNEL_ENCAP */
		u32			csum_flags;	            /* FLOW_ACTION_CSUM */
		u32			mark;		                /* FLOW_ACTION_MARK */
		u16     ptype;                  /* FLOW_ACTION_PTYPE */
		struct {				                /* FLOW_ACTION_QUEUE */
			u32		ctx;
			u32		index;
			u8		vf;
		} queue;
		struct {	                 			/* FLOW_ACTION_SAMPLE */
			struct psample_group	*psample_group;
			u32			rate;
			u32			trunc_size;
			bool			truncate;
		} sample;
		struct {				                /* FLOW_ACTION_POLICE */
			s64			burst;
			u64			rate_bytes_ps;
		} police;
		struct {				                /* FLOW_ACTION_CT */
			int action;
			u16 zone;
		} ct;
		struct {				                /* FLOW_ACTION_MPLS_PUSH */
			u32		label;
			__be16		proto;
			u8		tc;
			u8		bos;
			u8		ttl;
		} mpls_push;
		struct {				                /* FLOW_ACTION_MPLS_POP */
			__be16		proto;
		} mpls_pop;
		struct {			                 	/* FLOW_ACTION_MPLS_MANGLE */
			u32		label;
			u8		tc;
			u8		bos;
			u8		ttl;
		} mpls_mangle;
	};
};
```

```
struct flow_offload {
	struct flow_offload_tuple_rhash		tuplehash[FLOW_OFFLOAD_DIR_MAX];
	struct nf_conn				*ct;
	u16					flags;
	u16					type;
	u32					timeout;
	struct rcu_head				rcu_head;
};
```

```
struct flow_offload_tuple_rhash {
	struct rhash_head		         node;
	struct flow_offload_tuple	   tuple;
};
```

```
struct flow_match {
	struct flow_dissector	*dissector;
	void			*mask;
	void			*key;
};
```

```
struct flow_block_offload {
	enum flow_block_command command;
	enum flow_block_binder_type binder_type;
	bool block_shared;
	bool unlocked_driver_cb;
	struct net *net;
	struct flow_block *block;
	struct list_head cb_list;
	struct list_head *driver_block_list;
	struct netlink_ext_ack *extack;
};
```

```
struct flow_block_cb {
	struct list_head	driver_list;
	struct list_head	list;
	flow_setup_cb_t		*cb;
	void			*cb_ident;
	void			*cb_priv;
	void			(*release)(void *cb_priv);
	unsigned int		refcnt;
};
```

```
struct flow_rule {
	struct flow_match	match;
	struct flow_action	action;
};
```


* `nf_flow_table_offload.c`
```
struct nf_flow_match {
	struct flow_dissector	dissector;
	struct nf_flow_key	key;
	struct nf_flow_key	mask;
};

struct nf_flow_rule {
	struct nf_flow_match	match;
	struct flow_rule	*rule;
};
```

### NF INET initialization
```
static struct nf_flowtable_type flowtable_inet = {
	.family		= NFPROTO_INET,
	.init		  = nf_flow_table_init,
	.setup		= nf_flow_table_offload_setup,
	.action		= nf_flow_rule_route_inet,
	.free		  = nf_flow_table_free,
	.hook		  = nf_flow_offload_inet_hook,
	.owner		= THIS_MODULE,
};

__init nf_flow_inet_module_init(&flowtable_inet)   // nf_flow_table_inet.c
  nft_register_flowtable_type                      //nf_tables_api.c
    list_add_tail_rcu(struct nf_flowtable_type.list, &nf_tables_flowtables) //nf_tables_flowtables is the LIST_HEAD
```

* `nf_flow_table_init` in `nf_flow_table_core.c`
```
nf_flow_table_init
  INIT_DEFERRABLE_WORK    // workqueue
  flow_block_init
    INIT_LIST_HEAD(&flow_block->cb_list)  // struct flow_block is just a list
  rhashtable_init     // Resizable, Scalable, Concurrent Hash Tables
  queue_delayed_work   ==>  queue_delayed_work_on // workqueue.c
    __queue_delayed_work
  list_add(struct nf_flowtable->list, &flowtables) // flowtables is a LIST_HEAD
```

* `nf_flow_table_offload_setup`
```
nf_flow_table_offload_setup
   dev->netdev_ops->ndo_setup_tc
   nf_flow_table_block_setup
```

* `nf_flow_rule_route_inet`
It is responsible for searching for destination IP and device
```
nf_flow_rule_route_inet
  nf_flow_rule_route_ipv4 or nf_flow_rule_route_ipv6
    flow_offload_ipv4_snat or flow_offload_ipv4_dnat or flow_offload_port_snat or flow_offload_port_dnat
      struct flow_action_entry = flow_action_entry_next(flow_rule)
        flow_rule->rule->action.entries[flow_rule->rule->action.num_entries++]
      fill in struct flow_action_entry
    flow_offload_ipv4_checksum
    flow_offload_redirect
      flow->tuplehash[dir].tuple.dst_cache
      entry->dev = rt->dst.dev
```

* `nf_flow_offload_inet_hook`
It is responsible for searching for next neighbour, layer 2, and sending it out
```
nf_flow_offload_inet_hook
  nf_flow_offload_ip_hook // nf_flow_table_ip.c or nf_flow_offload_ipv6_hook  
    flow_offload_lookup   // nf_flow_table_core.c
      rhashtable_lookup( tuple )
    nf_flow_state_check    // if TPC FIN/RESET flow_offload_teardown()
    nf_flow_offload_dst_check
    nf_flow_nat_ip
    rt_nexthop(rt, flow->tuplehash[!dir].tuple.src_v4.s_addr)   // gateway or daddr
    neigh_xmit
      see routing.md
```
