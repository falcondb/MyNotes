## Net filter

### Key data structures
The book _Linux Netfilter Hacking How-to_ has more detailed explanations about the structs

* `netfilter/x_tables.h`

```
struct xt_table {
	struct list_head list;
	unsigned int valid_hooks;
	struct xt_table_info *private;
	u_int8_t af;		/* address/protocol family */
	int priority;		/* hook order */
	int (*table_init)(struct net *net);
};
```

```
struct xt_table_info {
	unsigned int size, number, initial_entries;
	unsigned int hook_entry[NF_INET_NUMHOOKS];
	unsigned int underflow[NF_INET_NUMHOOKS];
	unsigned int stacksize;
	void ***jumpstack;     // first dimension per cpu
	unsigned char entries[0] __aligned(8);   /* where the real data is placed  */
};
```


```
struct xt_target {
	struct list_head list;
	unsigned int (*target)(struct sk_buff *skb, const struct xt_action_param *);
	int (*checkentry)(const struct xt_tgchk_param *);
	void (*destroy)(const struct xt_tgdtor_param *);
	unsigned int hooks;
	unsigned short proto;
	unsigned short family;
};
```


```
struct xt_match {
	struct list_head list;
	bool (*match)(const struct sk_buff *skb, struct xt_action_param *);
	int (*checkentry)(const struct xt_mtchk_param *);
	void (*destroy)(const struct xt_mtdtor_param *);
	unsigned int hooks;
	unsigned short proto;
	unsigned short family;
};

```

```
struct xt_action_param {
	union {
		const struct xt_match *match;
		const struct xt_target *target;
	};
	union {
		const void *matchinfo, *targinfo;
	};
	const struct nf_hook_state *state;
	int fragoff;
	unsigned int thoff;
	bool hotdrop;
};
```

```
// For kernel
struct xt_entry_match
  struct xt_target
struct xt_entry_target
  struct xt_target
```

* `uapi/linux/netfileter_ipv4/ip_tables.h`
```
struct ipt_entry {
	struct ipt_ip ip; 	// src, dst, smsk, dmsk, in/outiface, in/outface_mask, proto
	unsigned int nfcache;    	/* Mark with fields that we care about. */
	__u16 target_offset;     	/* Size of ipt_entry + matches */
	__u16 next_offset;       	/* Size of ipt_entry + matches + target */
	unsigned int comefrom;   	/* Back pointer */
	struct xt_counters counters;
	unsigned char elems[0];  	/* The matches (if any), then the target, where the real data is placed */
};

```

* `netfilter.h`
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
	nf_hookfn		*hook;      	/* User fills in from here down. */
	struct net_device	*dev;
	void			*priv;
	u_int8_t		pf;
	unsigned int		hooknum;
	int			priority;        	/* Hooks are ordered in ascending priority. */
};
```


### net/netfilter
#### x_tables.c
```
for each in struct xt_target[]
  xt_register_target

```

#### core.c
```
nf_hook_entries_insert_raw
  nf_hook_entries_grow
  hooks_validate
  rcu_assign_pointer
```


### net/ipv4/

#### ip_tables.c
##### initialization

```
__init ip_tables_init
  register_pernet_subsys  // net/core/namespace.c
     register_pernet_operations  ==> __register_pernet_operations
        Add to list. TOBESTUDIED
  xt_register_targets
    list_add(&target->list, &xt[target->family].target)
    // ipv4 has two targets: XT_STANDARD_TARGET & XT_ERROR_TARGET in ip_tables.c
  xt_register_matches
    xt_register_match
      list_add(&match->list, &xt[match->family].match)
      // ipv4 has one match: IPPROTO_ICMP?
  nf_register_sockopt
    list_for_each_entry in nf_sockopts
      list_add(&struct nf_sockopt_ops->list, &nf_sockopts);
```

* `icmp_match` for ipv4's `struct xt_match`
```
icmp_match
  skb_header_pointer
  icmp_type_code_match
```


```
__exit ip_tables_fini
	nf_unregister_sockopt
  xt_unregister_matches
	xt_unregister_targets
	unregister_pernet_subsys
```

* `ipt_register_table`

```
 ipt_register_table
    translate_table     // Checks and translates the user-supplied table segment
      see translate_table section
    xt_register_table
      list_add(struct xt_table.list, struct net.xt.tables[struct xt_table.af]);
    nf_register_net_hooks     // net_filter/core.c
      nf_register_net_hook
        __nf_register_net_hook  for ipv4 and ipv6
          pp = nf_hook_entry_head
          new_hooks = nf_hook_entries_grow(pp, struct nf_hook_ops)
          rcu_assign_pointer(*pp, new_hooks)
```
`ipt_register_table` is called in `iptable_nat.c`, `iptable_mangle.c`, `iptable_raw.c`, `iptable_filter.c` and `iptable_security.c`

  * `iptable_nat_init`
    Why `iptable_nat_init` initialization of the hooks is different from the other tables? Is it because nat has the five predetermined hooks?
    ```
    __init iptable_nat_init
      register_pernet_subsys
      iptable_nat_table_init
        ipt_alloc_initial_table  ==>  xt_alloc_initial_table
        ipt_register_table
        ipt_nat_register_lookups
          nf_nat_ipv4_register_fn (nf_nat_ipv4_ops[])  // iptable_nat_do_chain, NFPROTO_IPV4, NF_INET_PRE_ROUTING, NF_IP_PRI_NAT_DST...
            nf_nat_register_fn    // nf_nat_core.c
              nat_proto_net = struct nat_net.nat_proto_net[pf]   // struct nat_net == struct nf_nat_hooks_net nat_proto_net[NFPROTO_NUMPROTO]
              nat_ops = nat_proto_net->nat_hook_ops
              nf_hook_entries_insert_raw
                see core.c section
    ```  

  * `iptable_filter_net_init`
    ```
    __init iptable_filter_net_init
      iptable_filter_table_init
         ipt_alloc_initial_table  ==>  xt_alloc_initial_table
            kmalloc(GFP_KERNEL | __GFP_ZERO)
         ipt_register_table    
    ```

  * `iptable_raw_table_init`
    ```
    __init iptable_raw_init
      xt_hook_ops_alloc
        kcalloc(num_hooks, sizeof(struct nf_hook_ops), GFP_KERNEL)
        struct nf_hook_ops[i].hook = fn

      register_pernet_subsys
      iptable_raw_table_init
        ipt_alloc_initial_table
        ipt_register_table
    ```
  * `iptable_mangle.c`
    ```
    __init iptable_mangle_init
      xt_hook_ops_alloc
      register_pernet_subsys
      iptable_mangle_table_init
        ipt_alloc_initial_table
        ipt_register_table
    ```

##### `ipt_do_table`
The hooks in tables call `ipt_do_table`
```
ipt_do_table
  ip_packet_match  // find a matched IP in the hook entries
    struct xt_action_param.match->match(skb, xt_action_param)
    if !t->u.kernel.target->target    // standard target
      // update jumpstack and get struct ipt_entry from the offset in struct xt_standard_target?
    struct xt_entry_target->u.kernel.target->target(skb, )
    until xt_action_param.hotdrop == false
```

```
do_replace
  struct ipt_replace tmp
  copy_from_user(&tmp, user, sizeof(tmp)
  newinfo = xt_alloc_table_info(tmp.size)
  copy_from_user(loc_cpu_entry, user + sizeof(tmp), tmp.size)
  translate_table(net, newinfo, loc_cpu_entry, &tmp)
  __do_replace
    t = xt_request_find_table_lock
    xt_replace_table(t, ...)
      repace xt_table.private type struct xt_table_info. it handles all the synchronization issues.

```

##### `translate_table`
```
translate_table
  /* Figures out from what hook each rule can be called: returns 0 if
   there are loops.  Puts hook bitmask in comefrom. */
  mark_source_chains
    if XT_STANDARD_TARGET && t->verdict > 0  // a jump
      xt_find_jump_offset
        // binary search for the target offset
    else move to the next struct ipt_entry // fall through
