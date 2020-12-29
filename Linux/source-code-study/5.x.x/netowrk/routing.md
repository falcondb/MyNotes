## Routing & Forwarding-Information Base (FIB)

### Key data structures
* `route.h`
```
struct rtable {
	struct dst_entry	dst;
	int			rt_genid;
	unsigned int		rt_flags;
	__u16			rt_type;
	__u8			rt_is_input;
	__u8			rt_uses_gateway;
	int			rt_iif;
	u8			rt_gw_family;
	union {
		__be32		rt_gw4;
		struct in6_addr	rt_gw6;
	};
	u32			rt_mtu_locked:1, rt_pmtu:31;
	struct list_head	rt_uncached;
	struct uncached_list	*rt_uncached_list;
};
```

* `net/dst.h`
Protocol independent destination cache definitions
```
struct dst_entry {
	struct net_device       *dev;
	struct  dst_ops	        *ops;
	unsigned long		_metrics;
	unsigned long   expires;
	struct xfrm_state	*xfrm;
	int			(*input)(struct sk_buff *);
	int			(*output)(struct net *net, struct sock *sk, struct sk_buff *skb);
	unsigned short		flags;
	unsigned short		header_len;	/* more space at head required */
	unsigned short		trailer_len;	/* space to reserve at tail */
	atomic_t		__refcnt;	/* 64-bit offset 64 */
	int			__use;
	unsigned long		lastuse;
	struct lwtunnel_state   *lwtstate;
	struct rcu_head		rcu_head;
	short			error;
	short			__pad;
	__u32			tclassid;
};
```

* `ip_fib.h`
```
struct fib_table {
	struct hlist_node	tb_hlist;
	u32			tb_id;
	int			tb_num_default;
	struct rcu_head		rcu;
	unsigned long 		*tb_data;
	unsigned long		__data[0];
};

struct fib_result {
	__be32			prefix;
	unsigned char		prefixlen;
	unsigned char		nh_sel;
	unsigned char		type;
	unsigned char		scope;
	u32			tclassid;
	struct fib_nh_common	*nhc;
	struct fib_info		*fi;
	struct fib_table	*table;
	struct hlist_head	*fa_head;
};
```

* `net/fib_rules.h`
```
struct fib_rule {
	struct list_head	list;
	int			iifindex, oifindex;
	u32			mark, mark_mask, flags, table;
	u8			action, l3mdev, proto, ip_proto;
	u32			target;
	__be64			tun_id;
	struct fib_rule __rcu	*ctarget;
	struct net		*fr_net;
	refcount_t		refcnt;
	u32			pref;
	int			suppress_ifgroup, suppress_prefixlen;
	char			iifname[IFNAMSIZ], oifname[IFNAMSIZ];
	struct fib_kuid_range	uid_range;
	struct fib_rule_port_range	sport_range, dport_range;
	struct rcu_head		rcu;
};
```
* `net/ipv4/fib_rules.c`
```
struct fib4_rule {
	struct fib_rule		common;
	u8			dst_len, src_len;
	u8			tos;
	__be32	src, rcmaskn, dst,dstmask;
	u32			tclassid;
};
```

### Key functions
#### initialization
* `net/core/fib_rules.c`
```
__init fib_rules_init
  rtnl_register   \\ netlink for fib_nl_newrule, fib_nl_delrule & fib_nl_dumprule
  register_pernet_subsys(&fib_rules_net_ops)
  register_netdevice_notifier(&fib_rules_notifier)


```

```
static struct pernet_operations fib_rules_net_ops = {
	.init = fib_rules_net_init,
	.exit = fib_rules_net_exit,
};
```
```
fib_rules_net_init
	INIT_LIST_HEAD(&net->rules_ops);
	spin_lock_init(&net->rules_mod_lock);
```

* `net/ipv4/fib_frontend.c`

```
__init ip_fib_init
 fib_trie_init
 register_pernet_subsys(&fib_net_ops);
 register_netdevice_notifier(&fib_netdev_notifier);
 register_inetaddr_notifier(&fib_inetaddr_notifier);
 rtnl_register  \\ netlink for inet_rtm_newroute, inet_rtm_delroute, inet_dump_fib


 static struct pernet_operations fib_net_ops = {
 	.init = fib_net_init,
 	.exit = fib_net_exit,
 };

fib_net_init
  ip_fib_net_init
 	nl_fib_lookup_init
  fib_proc_init
```
* `net/ipv4/route.c`
```
__init ip_rt_init
```

#### key operations

* `ip_route_input`
`ip_rcv_finish()` invokes `ip_route_input()` to find a `dst_entry`

* `ip_route_input`

* `fib_lookup`
FIB query

* `fib_rules_event`

* `fib_rules_export`

* `net/core/fib_rules.c`

* `net/ipv4/fib_rules.c`