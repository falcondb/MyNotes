## Traffic control

### Key data structures

#### Queuing discipline
* `net/pkt_generic.h`

```
truct Qdisc {
	int 			(*enqueue)(struct sk_buff *skb, struct Qdisc *sch, struct sk_buff **to_free);
	struct sk_buff *	(*dequeue)(struct Qdisc *sch);
	unsigned int		flags;
	u32			limit;
	const struct Qdisc_ops	*ops;
	struct qdisc_size_table	__rcu *stab;
	struct hlist_node       hash;
	u32			handle;
	u32			parent;
	struct netdev_queue	*dev_queue;
	struct net_rate_estimator __rcu *rate_est;
	struct gnet_stats_basic_cpu __percpu *cpu_bstats;
	struct gnet_stats_queue	__percpu *cpu_qstats;
	int			padded;
	refcount_t		refcnt;

	struct sk_buff_head	gso_skb ____cacheline_aligned_in_smp;
	struct qdisc_skb_head	q;       // queue of skb
	struct gnet_stats_basic_packed bstats;
	seqcount_t		running;
	struct gnet_stats_queue	qstats;
	unsigned long		state;
	struct Qdisc            *next_sched;
	struct sk_buff_head	skb_bad_txq;
	spinlock_t		busylock ____cacheline_aligned_in_smp;
	spinlock_t		seqlock;
	bool			empty;
	struct rcu_head		rcu;
};
```

```
struct Qdisc_ops {
	struct Qdisc_ops	*next;
	const struct Qdisc_class_ops	*cl_ops;
	char			id[IFNAMSIZ];
	int			priv_size;
	unsigned int		static_flags;

	int 			(*enqueue)(struct sk_buff *skb, struct Qdisc *sch, struct sk_buff **to_free);
	struct sk_buff *	(*dequeue)(struct Qdisc *);
	struct sk_buff *	(*peek)(struct Qdisc *);

	int			(*init)(struct Qdisc *sch, struct nlattr *arg, struct netlink_ext_ack *extack);
	void			(*reset)(struct Qdisc *);
	void			(*destroy)(struct Qdisc *);
	int			(*change)(struct Qdisc *sch, struct nlattr *arg, struct netlink_ext_ack *extack);
	void			(*attach)(struct Qdisc *sch);
	int			(*change_tx_queue_len)(struct Qdisc *, unsigned int);

	int			(*dump)(struct Qdisc *, struct sk_buff *);
	int			(*dump_stats)(struct Qdisc *, struct gnet_dump *);

	void			(*ingress_block_set)(struct Qdisc *sch, u32 block_index);
	void			(*egress_block_set)(struct Qdisc *sch, u32 block_index);
	u32			(*ingress_block_get)(struct Qdisc *sch);
	u32			(*egress_block_get)(struct Qdisc *sch);

	struct module		*owner;
};

```

```
struct Qdisc_class_ops {
	unsigned int		flags;
	struct netdev_queue *	(*select_queue)(struct Qdisc *, struct tcmsg *);
  // bind a queuing discipline to a class
	int			(*graft)(struct Qdisc *, unsigned long cl, struct Qdisc *, struct Qdisc **,	struct netlink_ext_ack *extack);
	struct Qdisc *		(*leaf)(struct Qdisc *, unsigned long cl);
	void			(*qlen_notify)(struct Qdisc *, unsigned long);
	unsigned long		(*find)(struct Qdisc *, u32 classid);
	int			(*change)(struct Qdisc *, u32, u32, struct nlattr **, unsigned long *, struct netlink_ext_ack *);
	int			(*delete)(struct Qdisc *, unsigned long);
	void			(*walk)(struct Qdisc *, struct qdisc_walker * arg);
	struct tcf_block *	(*tcf_block)(struct Qdisc *sch, unsigned long arg, struct netlink_ext_ack *extack);
	unsigned long		(*bind_tcf)(struct Qdisc *, unsigned long, u32 classid);
	void			(*unbind_tcf)(struct Qdisc *, unsigned long);
	int			(*dump)(struct Qdisc *, unsigned long, struct sk_buff *skb, struct tcmsg*);
	int			(*dump_stats)(struct Qdisc *, unsigned long, struct gnet_dump *);
};
```

#### Filter
* `net/sch_generic.c`

```
struct tcf_proto {
	struct tcf_proto __rcu	*next;
	void __rcu		*root;
	int			(*classify)(struct sk_buff *, const struct tcf_proto *, struct tcf_result *);
	__be16			protocol;
	u32			prio;        // order filters
	void			*data;
	const struct tcf_proto_ops	*ops;
	struct tcf_chain	*chain;
	spinlock_t		lock;
	bool			deleting;
	refcount_t		refcnt;
	struct rcu_head		rcu;
	struct hlist_node	destroy_ht_node;
};
```

```
struct tcf_proto_ops {
	struct list_head	head;
	char			kind[IFNAMSIZ];

	int			(*classify)(struct sk_buff *, struct tcf_proto *, struct tcf_result *);
	int			(*init)(struct tcf_proto*);
	void		(*destroy)(struct tcf_proto *tp, bool rtnl_held, struct netlink_ext_ack *extack);
	void*		(*get)(struct tcf_proto*, u32 handle);    // map a handle of a filter element to an internal filter identification
	void		(*put)(struct tcf_proto *tp, void *f);    // invoked to unreference a filter
	int			(*change)(struct net *, struct sk_buff *,struct tcf_proto*, unsigned long, u32 handle,
                    struct nlattr **,	void **, bool, bool, struct netlink_ext_ack *);
	int			(*delete)(struct tcf_proto *tp, void *arg, bool *last, bool rtnl_held, struct netlink_ext_ack *);
	void		(*walk)(struct tcf_proto *tp, struct tcf_walker *arg, bool rtnl_held);   // invokes callback to get configuration & statistics
	int			(*reoffload)(struct tcf_proto *tp, bool add, flow_setup_cb_t *cb, void *cb_priv, struct netlink_ext_ack *extack);
	void		(*hw_add)(struct tcf_proto *tp, void *type_data);
	void		(*hw_del)(struct tcf_proto *tp, void *type_data);
	void		(*bind_class)(void *, u32, unsigned long);
	void *	(*tmplt_create)(struct net *net, struct tcf_chain *chain, struct nlattr **tca, truct netlink_ext_ack *extack);
	void		(*tmplt_destroy)(void *tmplt_priv);
	/* rtnetlink specific */
	int			(*dump)(struct net*, struct tcf_proto*, void *, struct sk_buff *skb, struct tcmsg*,bool);
	int			(*tmplt_dump)(struct sk_buff *skb, struct net *net, void *tmplt_priv);
	struct module		*owner;
	int			flags;
};
```


### Key functions

#### ingress
* `net/sched/sch_ingress.c`
```
__init ingress_module_init
  register_qdisc(&ingress_qdisc_ops)
  register_qdisc(&clsact_qdisc_ops)


static const struct Qdisc_class_ops ingress_class_ops = {
	.flags		=	QDISC_CLASS_OPS_DOIT_UNLOCKED,
	.leaf		=	ingress_leaf,
	.find		=	ingress_find,
	.walk		=	ingress_walk,
	.tcf_block	=	ingress_tcf_block,
	.bind_tcf	=	ingress_bind_filter,
	.unbind_tcf	=	ingress_unbind_filter,
};

static struct Qdisc_ops ingress_qdisc_ops __read_mostly = {
	.cl_ops			=	&ingress_class_ops,
	.id			=	"ingress",
	.priv_size		=	sizeof(struct ingress_sched_data),
	.static_flags		=	TCQ_F_CPUSTATS,
	.init			=	ingress_init,
	.destroy		=	ingress_destroy,
	.dump			=	ingress_dump,
	.ingress_block_set	=	ingress_ingress_block_set,
	.ingress_block_get	=	ingress_ingress_block_get,
	.owner			=	THIS_MODULE,
};


static const struct Qdisc_class_ops clsact_class_ops = {
	.flags		=	QDISC_CLASS_OPS_DOIT_UNLOCKED,
	.leaf		=	ingress_leaf,
	.find		=	clsact_find,
	.walk		=	ingress_walk,
	.tcf_block	=	clsact_tcf_block,
	.bind_tcf	=	clsact_bind_filter,
	.unbind_tcf	=	ingress_unbind_filter,
};

static struct Qdisc_ops clsact_qdisc_ops __read_mostly = {
	.cl_ops			=	&clsact_class_ops,
	.id			=	"clsact",
	.priv_size		=	sizeof(struct clsact_sched_data),
	.static_flags		=	TCQ_F_CPUSTATS,
	.init			=	clsact_init,
	.destroy		=	clsact_destroy,
	.dump			=	ingress_dump,
	.ingress_block_set	=	clsact_ingress_block_set,
	.egress_block_set	=	clsact_egress_block_set,
	.ingress_block_get	=	clsact_ingress_block_get,
	.egress_block_get	=	clsact_egress_block_get,
	.owner			=	THIS_MODULE,
};  
```


#### egress
* `net/sched/sched_api.c`
```
__init pktsched_init
  register_qdisc for bfifo and qfifo, mq_qdisc_ops
  rtnl_register for DISC and CLASS
```

```
register_qdisc

```

```
qdisc_graft     // a new queuing discipline should be attached to the traffic-control tree
```

#### filter
* `cls_api.c`
```
__init tc_filter_init
  alloc_ordered_workqueue("tc_filter_workqueue", 0)
  register_pernet_subsys

  rtnl_register
```

#### net/pkt_sched.h
```
qdisc_run
  qdisc_run_begin (struct Qdisc)
    __qdisc_run
      while qdisc_restart
        __netif_schedule
  qdisc_run_end

```

responsible for getting the next packet from the queue of the network device and sending it
```
qdisc_restart
```
