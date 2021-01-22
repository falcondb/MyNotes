## IPSec

### Key data structures
`net/xfrm.h`
```
struct xfrm_state {
  struct xfrm_id		id;
  struct { ... } props;

}
```

```
struct xfrm_policy {
  struct xfrm_selector	selector;    //the characteristics of the flow that the Policy matches
  u8			action;
  struct xfrm_tmpl       	xfrm_vec[XFRM_MAX_DEPTH];   // template associated with this Policy
}
```

```
struct xfrm_input_afinfo {
	unsigned int		family;
	int			(*callback)(struct sk_buff *skb, u8 protocol, int err);
}

struct xfrm_state_afinfo {
  unsigned int			family;
  unsigned int			proto;
  __be16				eth_proto;
  const struct xfrm_type		*type_map[IPPROTO_MAX];
  ...
}
```

`uapi/linux/xfrm.h`
```
struct xfrm_id {
	xfrm_address_t	daddr;
	__be32		spi;   // discern between two traffic streams where different encryption rules and algorithms RFC 2401
	__u8		proto;
};
```

`net/xfrm/xfrm_user.c`
```
static const struct xfrm_link {
	int (*doit)(struct sk_buff *, struct nlmsghdr *, struct nlattr **);
	int (*start)(struct netlink_callback *);
	int (*dump)(struct sk_buff *, struct netlink_callback *);
	int (*done)(struct netlink_callback *);
	const struct nla_policy *nla_pol;
	int nla_max;
} xfrm_dispatch[XFRM_NR_MSGTYPES] = {
	[XFRM_MSG_NEWSA       - XFRM_MSG_BASE] = { .doit = xfrm_add_sa        },
	[XFRM_MSG_DELSA       - XFRM_MSG_BASE] = { .doit = xfrm_del_sa        },
  ...
}
```

### Initialization

#### Netlink initialization
`xfrm_user.c`
```
xfrm_user_net_init
  struct netlink_kernel_cfg cfg = {
  .groups	= XFRMNLGRP_MAX,
  .input	= xfrm_netlink_rcv,}

  nlsk = netlink_kernel_create
  rcu_assign_pointer(net->xfrm.nlsk, nlsk)
```

```
xfrm_netlink_rcv
	netlink_rcv_skb(skb, &xfrm_user_rcv_msg)
```

```
xfrm_user_rcv_msg
  link = &xfrm_dispatch[nlh->nlmsg_type]

  nlmsg_parse

  link->doit
```

Example of the `XFRM_MSG`
```
xfrm_add_sa
  x = xfrm_state_construct

  xfrm_state_hold(x)   ==> refcount_inc_not_zero   ==>   atomic_add_unless

  xfrm_state_add  or   xfrm_state_update

  xfrm_audit_state_add

  km_state_notify

  xfrm_state_put
```


```
static struct xfrm_mgr netlink_mgr = {
	.notify		= xfrm_send_state_notify,
	.acquire	= xfrm_send_acquire,
	.compile_policy	= xfrm_compile_policy,
	.notify_policy	= xfrm_send_policy_notify,
	.report		= xfrm_send_report,
	.migrate	= xfrm_send_migrate,
	.new_mapping	= xfrm_send_mapping,
	.is_alive	= xfrm_is_alive,
};

__init xfrm_user_init
  register_pernet_subsys

  xfrm_register_km(&netlink_mgr)
    list_add_tail_rcu(&km->list, &xfrm_km_list)

```

#### protocol initialization
* `net/ipv4/xfrm4_protocol.c`
```
static const struct xfrm_input_afinfo xfrm4_input_afinfo = {
	.family		=	AF_INET,
	.callback	=	xfrm4_rcv_cb,
};

xfrm4_protocol_init
  xfrm_input_register_afinfo(&xfrm4_input_afinfo)
    rcu_assign_pointer(xfrm_input_afinfo[afinfo->family], afinfo)
```

#### AH Initialization
* `net/ipv4/ah4.c`
```
static const struct xfrm_type ah_type =
{
	.proto	     	= IPPROTO_AH,
	.flags		= XFRM_TYPE_REPLAY_PROT,
	.init_state	= ah_init_state,
	.destructor	= ah_destroy,
	.input		= ah_input,
	.output		= ah_output
};

static struct xfrm4_protocol ah4_protocol = {
	.handler	=	xfrm4_rcv,
	.input_handler	=	xfrm_input,
	.cb_handler	=	ah4_rcv_cb,
	.err_handler	=	ah4_err,
};
```

```
__init ah4_init
  xfrm_register_type(&ah_type, AF_INET)
    afinfo = xfrm_state_get_afinfo
    afinfo->type_map[type->proto] = type
  xfrm4_protocol_register(&ah4_protocol, IPPROTO_AH)
```

```
ah_input
  ah = (struct ip_auth_hdr *)skb->data
  
```

#### ingress common
```
__init xfrm_input_init
  init_dummy_netdev
  gro_cells_init

  for each cpu
    trans = &per_cpu(xfrm_trans_tasklet, i)
    __skb_queue_head_init(&trans->queue)
    tasklet_init(&trans->tasklet, xfrm_trans_reinject, trans)
```

```
xfrm_input
  // GRO offload case

  sp = secpath_set(skb)
  daddr = skb_network_header(skb) + XFRM_SPI_SKB_CB(skb)->daddroff

  do
    skb_sec_path  ==> secpath_set ==> skb->extensions + ext->offset[id] << 3
    xfrm_state_lookup

    skb->mark = xfrm_smark_get(skb->mark, x)
    sp->xvec[sp->len++] = x

    if (crypto_done)
      nexthdr = x->type_offload->input_tail(x, skb)
    else
      nexthdr = x->type->input(x, skb)

    xfrm_parse_spi  
  while !err

  xfrm_rcv_cb
    xfrm_input_afinfo *afinfo = xfrm_input_get_afinfo(family)
    afinfo->callback

  // GRO  
```

```
xfrm_state_lookup ==> __xfrm_state_lookup
  h = xfrm_spi_hash(net, daddr, spi, proto, family)
  hlist_for_each_entry_rcu(x, net->xfrm.state_byspi + h, byspi)
    // search for a match: family,  id.spi, id.proto, daddr
```

#### ingress ipv4

* `net/ipv4/xfrm4_protocol.c`

```
static const struct net_protocol esp4_protocol = {
	.handler	=	xfrm4_esp_rcv,
	.err_handler	=	xfrm4_esp_err,
};

static const struct net_protocol ah4_protocol = {
	.handler	=	xfrm4_ah_rcv,
	.err_handler	=	xfrm4_ah_err,
};

static const struct net_protocol ipcomp4_protocol = {
	.handler	=	xfrm4_ipcomp_rcv,
	.err_handler	=	xfrm4_ipcomp_err,
};
```


```
xfrm4_rcv_cb
  head = proto_handlers(protocol)
    IPPROTO_ESP:  esp4_handlers
    IPPROTO_AH:   ah4_handlers
    IPPROTO_COMP: ipcomp4_handlers
  for_each_protocol_rcu(*head, handler)
    handler->cb_handler
```

```
xfrm4_ah_rcv
  for_each_protocol_rcu(ah4_handlers, handler)
    handler->handler(skb)

```

* `net/ipv4/xfrm4_input.c`
```
xfrm4_rcv
  xfrm4_rcv_spi
    xfrm_input
```

#### ingress ipv6
* `net/ipv6/xfrm4_protocol.c`
```
static const struct xfrm_input_afinfo xfrm6_input_afinfo = {
	.family		=	AF_INET6,
	.callback	=	xfrm6_rcv_cb,
};
```
