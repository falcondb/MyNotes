## Routing
### Key data structure
#### net/route.h
```
struct rtable {
	struct dst_entry	dst;
	int			rt_genid;
	unsigned int		rt_flags;
	__u16			rt_type;
	__u8			rt_is_input;
	__u8			rt_uses_gateway;
	int			rt_iif;
	__be32			rt_gateway;				/* Info on neighbour */
	u32			rt_mtu_locked:1, rt_pmtu:31;			/* Miscellaneous cached information */
	struct list_head	rt_uncached;
	struct uncached_list	*rt_uncached_list;
};
```

#### net/neighbour.c
```
struct neighbour {
	struct neighbour __rcu	*next;
	struct neigh_table	*tbl;
	struct neigh_parms	*parms;
	unsigned long		confirmed, updated;
	rwlock_t		lock;
	refcount_t		refcnt;
	unsigned int		arp_queue_len_bytes;
	struct sk_buff_head	arp_queue;
	struct timer_list	timer;
	unsigned long		used;
	atomic_t		probes;
	__u8			flags, nud_state, type, dead;
	u8			protocol;
	seqlock_t		ha_lock;
	unsigned char		ha[ALIGN(MAX_ADDR_LEN, sizeof(unsigned long))] __aligned(8);
	struct hh_cache		hh;
	int			(*output)(struct neighbour *, struct sk_buff *);
	const struct neigh_ops	*ops;
	struct list_head	gc_list;
	struct rcu_head		rcu;
	struct net_device	*dev;
	u8			primary_key[0];
}
```

```
struct neigh_hash_table {
	struct neighbour __rcu	**hash_buckets;
	unsigned int		hash_shift;
	__u32			hash_rnd[NEIGH_NUM_HASH_RND];
	struct rcu_head		rcu;
};
```

```
struct neigh_table {
	int			family;
	unsigned int		entry_size;
	unsigned int		key_len;
	__be16			protocol;
	__u32		(*hash)					 (void *, struct net_device *, __u32 *);
	bool		(*key_eq)        (const struct neighbour *, const void *pkey);
	int			(*constructor)   (struct neighbour *);
	int			(*pconstructor)  (struct pneigh_entry *);
	void		(*pdestructor) 	 (struct pneigh_entry *);
	void		(*proxy_redo) 	 (struct sk_buff *skb);
	bool		(*allow_add)  	 (struct net_device *, struct netlink_ext_ack *);
	char			*id;
	struct neigh_parms	parms;
	struct list_head	parms_list;
	int			gc_interval, gc_thresh1, gc_thresh2, gc_thresh3;
	unsigned long		last_flush;
	struct delayed_work	gc_work;
	struct timer_list 	proxy_timer;
	struct sk_buff_head	proxy_queue;
	atomic_t		entries;
	atomic_t		gc_entries;
	struct list_head	gc_list;
	rwlock_t		lock;
	unsigned long		last_rand;
	struct neigh_statistics	__percpu *stats;
	struct neigh_hash_table __rcu *nht;
	struct pneigh_entry	**phash_buckets;
};
```

```
struct neigh_ops {
	int			family;
	void			(*solicit)(struct neighbour *, struct sk_buff *);
	void			(*error_report)(struct neighbour *, struct sk_buff *);
	int				(*output)(struct neighbour *, struct sk_buff *);
	int				(*connected_output)(struct neighbour *, struct sk_buff *);
};
```
### neighbour
#### initialization
```
__init neigh_init
  rtnl_register(PF_UNSPEC, RTM_NEWNEIGH, neigh_add, NULL, 0);   // net/core/rtnetlin.c, the netlink socket
    rtnl_register_internal
      tab = rtnl_msg_handlers[protocol]         // rtnl_msg_handlers, a static array in rtnetlin.c
      kmemdup or kzalloc for struct rtnl_link
      rcu_assign_pointer(tab[msgindex], link);
  rtnl_register(PF_UNSPEC, RTM_DELNEIGH, neigh_delete, NULL, 0);
  rtnl_register(PF_UNSPEC, RTM_GETNEIGH, neigh_get, neigh_dump_info, 0);
  rtnl_register(PF_UNSPEC, RTM_GETNEIGHTBL, NULL, neightbl_dump_info, 0);
  rtnl_register(PF_UNSPEC, RTM_SETNEIGHTBL, neightbl_set, NULL, 0);
```

```
neigh_find_table
  AF_INET   ==>  neigh_tables[NEIGH_ARP_TABLE]      //neigh_tables is a static array in neighbour.c
  AF_INET6  ==>  neigh_tables[NEIGH_ND_TABLE]
  AF_DECnet ==>  neigh_tables[NEIGH_DN_TABLE]
```
MPLS uses its own table, `NEIGH_LINK_TABLE`

```
# net/core/neighbour.c
neigh_xmit
  if NEIGH_ARP_TABLE for ipv4
    __ipv4_neigh_lookup_noref
  else  for AF_INET6 and DECnet (or may not anymore)
    __neigh_lookup_noref
  neigh->output  
  if NEIGH_LINK_TABLE for MPLS
    dev_hard_header
    dev_queue_xmit
```



```
neigh_timer_handler     // a timer expires for a neighbour entry
or neigh_add / neigh_add / neigh_delete ==> __neigh_update
  neigh_connect
    neigh->output = neigh->ops->connected_output
```
