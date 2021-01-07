## NetFilter

### Key data structures

* `linux/netfilter.h`
```
struct nf_hook_state {
	unsigned int hook;
	u_int8_t pf;
	struct net_device *in;
	struct net_device *out;
	struct sock *sk;
	struct net *net;
	int (*okfn)(struct net *, struct sock *, struct sk_buff *);
};
```
```
struct nf_hook_ops {
	nf_hookfn		        *hook;        	/* User fills in from here down. */
	struct net_device	  *dev;
	void			          *priv;
	u_int8_t		        pf;
	unsigned int		    hooknum;
	int			            priority;
};
```

```
struct nf_hook_entry {
	nf_hookfn			*hook;
	void				*priv;
};

struct nf_hook_entries {
	u16				num_hook_entries;
	struct nf_hook_entry		hooks[];
};
```

```
typedef unsigned int nf_hookfn(void *priv, struct sk_buff *, struct nf_hook_state *)
```

```
struct nf_sockopt_ops {
	struct list_head list;
	u_int8_t pf;
	int set_optmin,set_optmax,get_optmin,get_optmax;
	int (*set)(struct sock *sk, int optval, void __user *user, unsigned int len);
	int (*get)(struct sock *sk, int optval, void __user *user, int *len);
	struct module *owner;
};
```

* `uapi/linux/netfilter.h`
```
* Responses from hook functions. */
enum nf_inet_hooks {5 hooks here}

union nf_inet_addr {
	__u32		all[4];
	__be32		ip;
	__be32		ip6[4];
	struct in_addr	in;
	struct in6_addr	in6;
};

```
* `uapi/linux/netfilter_ipv4.h`
```
/* IP Hooks DEFINE*/
enum nf_ip_hook_priorities {...}

```

* `net/netfilter/nf_conntrack_tuple.h`
```
struct nf_conntrack_tuple {    /* This contains the information to distinguish a connection. */
	struct nf_conntrack_man src;
	struct {
		union nf_inet_addr u3;
		union {     			/* Add other protocols here. */
			__be16 all;
			struct {			__be16 port;       	  		} tcp;
			struct {      __be16 port;           		} udp;
			struct {   		u_int8_t type, code;			} icmp;
			struct {			__be16 port;              } dccp;
			struct {			__be16 port;        			} sctp;
			struct {			__be16 key;         			} gre;
		} u;
		u_int8_t protonum;
		u_int8_t dir;     		/* The direction (for tuplehash) */
	} dst;
};

struct nf_conntrack_man {
	union nf_inet_addr u3;
	union nf_conntrack_man_proto u;
	u_int16_t l3num;
};

struct nf_conntrack_tuple_hash {
	struct hlist_nulls_node hnnode;
	struct nf_conntrack_tuple tuple;
};

```

* `net/netfilter/nf_conntrack.h`
```
struct nf_conn {
	struct nf_conntrack ct_general;
	spinlock_t	lock;
	u32 timeout;
	struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];
	unsigned long status;
	u16		cpu;
	possible_net_t ct_net;
	struct hlist_node	nat_bysource;
	u8 __nfct_init_offset[0];
	struct nf_conn *master;
	u_int32_t mark;
	u_int32_t secmark;
	struct nf_ct_ext *ext;
	union nf_conntrack_proto proto;    	/* Storage reserved for other modules, must be the last member */
};
```

* `uapi/linux/netfilter/nf_conntrack_common.h`
```
enum ip_conntrack_info
enum ip_conntrack_status
enum ip_conntrack_events
```

* `nf_conntrack_l4proto.h`
```
struct nf_conntrack_l4proto {
	u_int8_t l4proto;
	u16 nlattr_size;                                     	/* protoinfo nlattr size, closes a hole */
	bool (*can_early_drop) (const struct nf_conn *ct);    	/* called by gc worker if table is full */
	int (*to_nlattr)       (struct sk_buff *, struct nlattr *, struct nf_conn *); 	/* convert protoinfo to nfnetink attributes */
	int (*from_nlattr)     (struct nlattr *tb[], struct nf_conn *ct);   	        /* convert nfnetlink attributes to protoinfo */
	int (*tuple_to_nlattr) (struct sk_buff *, struct nf_conntrack_tuple *);
	unsigned int (*nlattr_tuple_size)  (void);
	int (*nlattr_to_tuple) (struct nlattr *tb[], struct nf_conntrack_tuple *t);
	const struct nla_policy *nla_policy;
	struct {
		int (*nlattr_to_obj)  (struct nlattr *tb[], struct net *net, void *data);
		int (*obj_to_nlattr)  (struct sk_buff *skb, const void *data);
		u16 obj_size, nlattr_max;
		const struct nla_policy *nla_policy;
	} ctnl_timeout;
	void (*print_conntrack)(struct seq_file *s, struct nf_conn *);   	/* Print out the private part of the conntrack. */
};
```
`struct nlattr` is defined in `uapi/linux/netlink.h`, the NetLink message.


* `net/netfilter/nf_conntrack_proto.c`
```
static const struct nf_hook_ops ipv4_conntrack_ops[] = {
	{
		.hook		= ipv4_conntrack_in,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK,
	},
	{
		.hook		= ipv4_conntrack_local,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_CONNTRACK,
	},
	{
		.hook		= ipv4_confirm,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM,
	},
	{
		.hook		= ipv4_confirm,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM,
	},
};
```

```
static const struct nf_hook_ops ipv6_conntrack_ops[] = {
	{
		.hook		= ipv6_conntrack_in,
		.pf		= NFPROTO_IPV6,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP6_PRI_CONNTRACK,
	},
	{
		.hook		= ipv6_conntrack_local,
		.pf		= NFPROTO_IPV6,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP6_PRI_CONNTRACK,
	},
	{
		.hook		= ipv6_confirm,
		.pf		= NFPROTO_IPV6,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP6_PRI_LAST,
	},
	{
		.hook		= ipv6_confirm,
		.pf		= NFPROTO_IPV6,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP6_PRI_LAST - 1,
	},
};

```


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

### NetFilter connection tracking initialization
```
__init nf_conntrack_standalone_init
  nf_conntrack_init_start
    nf_ct_alloc_hashtable
    kmem_cache_create( struct nf_conn )
  register_pernet_subsys(&nf_conntrack_net_ops)	.init = nf_conntrack_pernet_init	.exit_batch	= nf_conntrack_pernet_exit,
    nf_conntrack_pernet_init
      nf_conntrack_standalone_init_sysctl
      nf_conntrack_standalone_init_proc
        proc_create_net( ct_seq_ops )   ==>  proc_create_net_data
          proc_create_reg
            __proc_create  // create struct proc_dir_entry
          proc_register    // link parent directory and the dir_entry
        proc_create_net( ct_cpu_seq_ops )
      nf_conntrack_init_net
        nf_conntrack_expect_pernet_init
        nf_conntrack_acct_pernet_init
        nf_conntrack_tstamp_pernet_init
        nf_conntrack_ecache_pernet_init
        nf_conntrack_helper_pernet_initki
        nf_conntrack_proto_pernet_init
          nf_conntrack_generic_init_net
            nf_ct_netns_do_get(net, NFPROTO_IPV4 / 6)   // usage accounting
          nf_conntrack_udp_init_net
          nf_conntrack_tcp_init_net
          nf_conntrack_icmp_init_net
      nf_ct_netns_get
        nf_ct_netns_inet_get
          NFPROTO_IPV4:
            nf_defrag_ipv4_enable ==> nf_register_net_hooks( ipv4_defrag_ops )
            nf_register_net_hooks ==> nf_register_net_hook  ==> __nf_register_net_hook
            see iptables.md
          NFPROTO_BRIDGE:
            nf_register_net_hooks

    nf_conntrack_pernet_exit
  nf_conntrack_init_end


```

```
static const struct seq_operations ct_seq_ops = {
	.start = ct_seq_start,
	.next  = ct_seq_next,
	.stop  = ct_seq_stop,
	.show  = ct_seq_show
};


static const struct seq_operations ct_cpu_seq_ops = {
	.start	= ct_cpu_seq_start,
	.next	= ct_cpu_seq_next,
	.stop	= ct_cpu_seq_stop,
	.show	= ct_cpu_seq_show,
};

ct_cpu_seq_start
  struct ct_iter_state *st = seq->private
  nf_conntrack_get_ht(&st->hash, &st->htable_size); // get the hash and table size
  ct_get_idx( pos ) // get list node at offset pos

ct_seq_next
  (*pos)++;
  ct_get_next

ct_cpu_seq_show
  seq_printf
```

### NetFilter Flow initialization

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

### NetFilter hooks
* `ipv4_conntrack_in` in `net/netfilter/nf_conntrack_proto.c`
```
ipv4_conntrack_in   ==>   nf_conntrack_in   #net/netfiler/nf_conntrack_core.c
  nf_ct_get     // get struct nf_conn, and ip_conntrack_info
    skb_get_nfct  ==>  skb->_nfct
      // the lowest 3 bits of skb->_nfct is the value of ip_conntrack_info, the rest higher value is the address of struct nf_conn
  if ICMP   nf_conntrack_handle_icmp  checks ICMP header then checks IP header
    nf_conntrack_inet_error     #net/netfiler/nf_conntrack_proto_icmp.c
      nf_ct_get_tuplepr  ==>  nf_ct_get_tuple
        // populate struct nf_conntrack_tuple from skb
      nf_ct_invert_tuple
        // invert the src and dst IPs and ports
      nf_conntrack_find_get(innertuple)  // the ct of the inverted tuple
        rcu_read_lock
        __nf_conntrack_find_get
          ____nf_conntrack_find   // search for the connection hashtable
          // check if the tuples are equal and connection is confirmed, etc.
      // check if the addresses of the two directions match
      // update skb's ip_conntrack_info

  resolve_normal_ct   // create a new tuple
    nf_ct_get_tuple
    __nf_conntrack_find_get
    init_conntrack  if not found
      __nf_conntrack_alloc ==>  kmem_cache_alloc
      nf_ct_add_to_unconfirmed_list
    // update skb's ip_conntrack_info

```
* `ipv4_conntrack_local`
```
ipv4_conntrack_local
  nf_conntrack_in
```

* `ipv4_confirm`
```
ipv4_confirm
  nf_conntrack_confirm
    if !nf_ct_is_confirmed
      __nf_conntrack_confirm
        __nf_conntrack_hash_insert
          hlist_nulls_add_head_rcu(&ct->tuplehash[IP_CT_DIR_ORIGINAL]
          hlist_nulls_add_head_rcu(&ct->tuplehash[IP_CT_DIR_REPLY]  

      nf_ct_deliver_cached_events
  nf_confirm
    if IPS_SEQ_ADJUST_BIT       // TCP sequence adjusted
      nf_ct_seq_adjust          // adjust TCP send seq and ack seq
    nf_conntrack_confirm
```

### Conntrack defragment
#### data structures
* `net/ipv4/netfilter/nf_defrag_ipv4.c`
```
static const struct nf_hook_ops ipv4_defrag_ops[] = {
	{
		.hook		= ipv4_conntrack_defrag,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK_DEFRAG,
	},
	{
		.hook           = ipv4_conntrack_defrag,
		.pf             = NFPROTO_IPV4,
		.hooknum        = NF_INET_LOCAL_OUT,
		.priority       = NF_IP_PRI_CONNTRACK_DEFRAG,
	},
};
```
#### Execution



### Netfilter NAT

#### Key data structures
* `net/netfileter/nf_nat_l3proto.h`
```
struct nf_nat_l3proto {
	u8	l3proto;
	bool	(*manip_pkt)    (...);
	void	(*csum_update)  (...);
	void	(*csum_recalc)  (...);
	void	(*decode_session)   (...);
	int	  (*nlattr_to_range)  (...);
};
```


* `net/netfilter/nf_nat_core.c`
```
struct nf_nat_hooks_net {
	struct nf_hook_ops *nat_hook_ops;
	unsigned int users;
};


static struct nf_nat_hook nat_hook = {
	.parse_nat_setup	= nfnetlink_parse_nat_setup,
	.decode_session		= __nf_nat_decode_session,
	.manip_pkt		= nf_nat_manip_pkt,
};
```

* `net/netfilter/nf_nat_proto.c`
```
static const struct nf_hook_ops nf_nat_ipv4_ops[] = {
	/* Before packet filtering, change destination */
	{
		.hook		= nf_nat_ipv4_in,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_NAT_DST,
	},
	/* After packet filtering, change source */
	{
		.hook		= nf_nat_ipv4_out,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_NAT_SRC,
	},
	/* Before packet filtering, change destination */
	{
		.hook		= nf_nat_ipv4_local_fn,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_NAT_DST,
	},
	/* After packet filtering, change source */
	{
		.hook		= nf_nat_ipv4_fn,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_NAT_SRC,
	},
};
```

#### NAT initialization
```
__init nf_nat_init
  nf_ct_alloc_hashtable
  nf_ct_extend_register
  register_pernet_subsys( nat_net_ops )
  nf_ct_helper_expectfn_register
```

#### NAT functions

* `nfnetlink_parse_nat_setup`

```
nfnetlink_parse_nat_setup
  __nf_nat_alloc_null_binding  
    nf_nat_setup_info
      // update the
      get_unique_tuple
      nf_ct_invert_tuple(&reply, &new_tuple)
      nf_conntrack_alter_reply(ct, &reply)

      if NF_NAT_MANIP_SRC
        hash_by_src(..., &ct->tuplehash[IP_CT_DIR_ORIGINAL].tuple)
        hlist_add_head_rcu(&ct->nat_bysource, &nf_nat_bysource[srchash])

```
Update the `ct->tuplehash[IP_CT_DIR_REPLY]` in `struct nf_conn` if necessry; Add to `struct nf_conn->nat_bysource` for _SNAT_

* `nf_nat_manip_pkt`
```
// nf_nat_proto.c
nf_nat_manip_pkt
  nf_ct_invert_tuple(&target, &ct->tuplehash[!dir].tuple)
  nf_nat_ipv4_manip_pkt   or   nf_nat_ipv6_manip_pkt

```

```
nf_nat_ipv4_manip_pkt
  l4proto_manip_pkt
    xxx_manip_pkt  // the xxx_manip_pkt for different protocols
      // tcp_manip_pkt ==> src dst port swap, nf_csum_update, inet_proto_csum_replace2

  csum_replace4   // checksum.h
```

```
nf_nat_ipv6_manip_pkt
  ipv6_skip_exthdr
  l4proto_manip_pkt
  swap src and dst IPs
```


* `nf_nat_ipv4_in`
Before packet filtering, change destination, NF_INET_PRE_ROUTING
```
nf_nat_ipv4_in      // if daddr is changed, skb_dst_drop
  nf_nat_ipv4_fn    // see nf_nat_ipv4_fn section
```

* `nf_nat_ipv4_out`
After packet filtering, change source, NF_INET_POST_ROUTING
```
nf_nat_ipv4_out
  nf_nat_ipv4_fn  // see nf_nat_ipv4_fn section
  XFRM stff
```    


* `nf_nat_ipv4_local_fn`
Before packet filtering, change destination, NF_INET_LOCAL_OUT
```
nf_nat_ipv4_local_fn
  nf_nat_ipv4_fn  // See nf_nat_ipv4_fn section
    // Accept? return
  if tuplehash[dir].tuple.dst.u3.ip != tuplehash[!dir].tuple.src.u3.ip  
      ip_route_me_harder
        ip_route_output_key  ==>  ip_route_output_flow
          __ip_route_output_key

          if flp4->flowi4_proto     xfrm_lookup_route
        skb_dst_drop
        skb_dst_set(skb, &rt->dst)  ==>  skb->_skb_refdst = rt->dst

```

* `nf_nat_ipv4_fn`
After packet filtering, change source, NF_INET_LOCAL_IN
```
nf_nat_ipv4_fn
  if ICMP   nf_nat_icmp_reply_translation
  nf_nat_inet_fn
```

* `ipt_snat_target` & `ipt_dnat_target()`
```
ipt_snat_target
```

```
ipt_dnat_target
```

#### NAT Common

* `nf_nat_inet_fn` in  `nf_nat_core.c`
Enter NAT. Priorities at these hooking points: _Conntrack_ > _NAT_ > _Packet Filtering_. _Conntrack_ has a higher priority than NAT, since NAT relies on the results of connection tracking.
```
nf_nat_inet_fn
  nfct_nat  ==>   nf_ct_ext_find  // get the extension at offset NF_CT_EXT_NAT
  if !nf_nat_initialized
    for each hook in lpriv->entries
    lpriv->entries->->hooks[i].hook(e->hooks[i].priv, skb, state)
    nf_nat_alloc_null_binding     // see nf_nat_alloc_null_binding here
  else    // handle NF_NAT_MASQUERADE cases

  nf_nat_packet
    nf_nat_manip_pkt    // nf_nat_manip_pkt section
```
If compare `nf_nat_ipv4_in` with `nf_nat_manip_pkt`, `nf_nat_ipv4_in` calss all the nat hooks registered to finalize the src and dst IPs. After the IPs are finalized, allocate the `nf_conntrack_tuple` in (`nf_nat_alloc_null_binding` ==> `nf_nat_setup_info`), revert the direction in `nf_conn`, and update the IPs in `skb`
