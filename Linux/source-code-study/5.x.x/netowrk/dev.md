## Network device

### Key data structues
#### `netdevice.h`
* `struct net_device`
  see network/key-data-structure.md

* `softnet_data`
Incoming packets are placed on per-CPU queues
```
struct softnet_data {
	struct list_head	poll_list;
	struct sk_buff_head	process_queue;
	unsigned int		processed, time_squeeze, received_rps; // rps: Receive Packet Steering
	struct sd_flow_limit __rcu *flow_limit;
	struct Qdisc		*output_queue;
	struct Qdisc		**output_queue_tailp;
	struct sk_buff	*completion_queue;
	struct sk_buff_head	xfrm_backlog;
	struct {
		u16 recursion;
		u8  more;
	} xmit;
	unsigned int		dropped;
	struct sk_buff_head	input_pkt_queue;
	struct napi_struct	backlog;
};
```

* `napi_struct`
Structure for NAPI scheduling similar to tasklet but with weighting.
New API, it defers incoming message handling until there is a sufficient amount of them so that it is worth handling them all at once
```
struct napi_struct {
	struct list_head	poll_list;
	unsigned long		state;
	int			weight;
	unsigned long		gro_bitmask;
	int			(*poll)(struct napi_struct *, int);
	int			poll_owner;
	struct net_device	*dev;
	struct gro_list		gro_hash[GRO_HASH_BUCKETS];
	struct sk_buff		*skb;
	struct list_head	rx_list; /* Pending GRO_NORMAL skbs */
	int			rx_count; /* length of rx_list */
	struct hrtimer		timer;
	struct list_head	dev_list;
	struct hlist_node	napi_hash_node;
	unsigned int		napi_id;
};

```

```
struct packet_type {
	__be16			type;	       /* This is really htons(ether_type). */
	bool			  ignore_outgoing;
	struct net_device	*dev;	 /* NULL is wildcarded here */
	int			  (*func)      (struct sk_buff *, struct net_device *, struct packet_type *, struct net_device *);
	void			(*list_func) (struct list_head *, struct packet_type *, struct net_device *);
	bool			(*id_match)  (struct packet_type *ptype, struct sock *sk);
	void			*af_packet_priv;
	struct list_head	list;
};
```

#### net/core/dev.c
```
struct list_head ptype_base[PTYPE_HASH_SIZE] __read_mostly;
struct list_head ptype_all __read_mostly;
```


### Key functions
#### Initialization
```
__init net_dev_init
  // TOBESTUDIED
  open_softirq(NET_TX_SOFTIRQ, net_tx_action)
  open_softirq(NET_RX_SOFTIRQ, net_rx_action)  

```

#### Key functions
* `netif_receive_skb`

```
netif_receive_skb
  netif_receive_skb_internal
    __netif_receive_skb
      __netif_receive_skb_one_core
        __netif_receive_skb_core
```
More direct receive version of netif_receive_skb().  It should only be used by callers that have a need to skip RPS and Generic XDP.
Caller must also take care of handling if (page_is_)pfmemalloc.
This function may only be called from softirq context and interrupts should be enabled.
```
netif_receive_skb_core
  __netif_receive_skb_one_core
    __netif_receive_skb_core
```

```
__netif_receive_skb_core
  do_xdp_generic
  deliver_skb
    struct packet_type ->func // see dev_add_pack

  sch_handle_ingress
  nf_ingress

  vlan cases:
  vlan_do_receive
```

```
// net/8021q/vlan_core.c
vlan_do_receive
  vlan_dev = vlan_find_dev(skb->dev, vlan_proto, vlan_id)
  skb->dev = vlan_dev
```

* `net_tx_action`
the handling routine of `NET_RX_SOFTIRQ`
```
net_tx_action
  free skb in sd->completion_queue
  for each Qdisc in sd->output_queue
    qdisc_run

```
* `net_rx_action`
```
net_rx_action
  list_splice_init(&sd->poll_list, &list) // make the list to struct softnet_data->poll_list
  for list is not empty
    struct napi_struct *n = list_first_entry  // first element of struct napi_struct.poll_list
    napi_poll
      // napi_poll code block
  __raise_softirq_irqoff(NET_RX_SOFTIRQ)   // softirq.c

  net_rps_action_and_irq_enable
    local_irq_enable
      net_rps_send_ipi if rps is set
```

```
napi_poll
  list_del_init(&n->poll_list)
  if NAPI_STATE_SCHED
    n->poll
  // gro stuff, TOBESTUDIED
  list_add_tail(&n->poll_list, repoll)
```

* `dev_add_pack`
Add a protocol handler to the networking stack.
It registers with the Linux network architecture the layer-3 protocol represented by the `struct packet_type pt`
```
head = ptype_head(pt)
spin_lock
list_add_rcu(&pt->list, head)
```

#### Packat processing path
* `netif_rx` in net/core/dev.c
completes the interrupt handling
```
netif_rx  ==>   netif_rx_internal
  #ifdef CONFIG_RPS
    cpu = get_rps_cpu
    enqueue_to_backlog
  #endif
    enqueue_to_backlog
        __skb_queue_tail(&sd->input_pkt_queue, skb)
```

* `dev_queue_xmit`
Used by protocol instances of the higher protocols to send a packet in the form of the socket buffer over a network device
```
dev_queue_xmit  ==> __dev_queue_xmit
  txq = netdev_core_pick_tx
  q = txq->qdisc
  if q->enqueue
    __dev_xmit_skb
      if sch_direct_xmit
        __qdisc_run
      else
        q->enqueue(skb,...)
```


* `__qdisc_run` in `net/sched/sch_generic.c`
```
__qdisc_run
  while qdisc_restart
    __netif_schedule  ==> __netif_reschedule
    softnet_data = this_cpu_ptr(&softnet_data);
    q->next_sched = NULL;
    *sd->output_queue_tailp = q;
    sd->output_queue_tailp = &q->next_sched;
    raise_softirq_irqoff(NET_TX_SOFTIRQ);

```

* `dev_open`

* `dev_close`


#### XDP
* `dev_change_xdp_fd`
called by `do_setlink` in `rtnetlink.c`
```
dev_change_xdp_fd
  bpf_op = generic_xdp_install
    rcu_assign_pointer(dev->xdp_prog, xdp->prog)
  prog = bpf_prog_get_type_dev  ==> __bpf_prog_get  ==> ____bpf_prog_get
    f.file->private_data
  dev_xdp_install(dev, bpf_op, ..., prog)
    xdp.prog = prog
    bpf_op(dev, &xdp)
```

* `do_xdp_generic`
```
do_xdp_generic
  netif_receive_generic_xdp
    see netif_receive_generic_xdp section
  XDP_REDIRECT:
    xdp_do_generic_redirect
      BPF_MAP_TYPE_DEVMAP:
        dev_map_generic_redirect
          xdp_ok_fwd_dev
          generic_xdp_tx
      BPF_MAP_TYPE_XSKMAP
        xsk_generic_rcv
          xsk_rcv
            __xsk_rcv
              xsk_copy_xdp(xsk_xdp, xdp)
              __xsk_rcv_zc
                xskq_prod_reserve_desc
  XDP_TX:
    generic_xdp_tx
```

```
generic_xdp_tx
  netdev_start_xmit   ==>   __netdev_start_xmit
    struct net_device_ops.ndo_start_xmit(skb, dev)

```

```
netif_receive_generic_xdp
  set xdp->data to the MAC addr, xdp->end to the end of header (before the package)
  bpf_prog_run_xdp
  handle bpf_xdp_adjust_head and bpf_xdp_adjust_tail, adjust skb and skb->mac_header
```
