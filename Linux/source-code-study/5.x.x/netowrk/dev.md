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
	struct sk_buff_head	input_pkt_queue;   // Device adds packets here
	struct napi_struct	backlog;    // for RPS
};
```

* `hh_cache`
```
struct hh_cache {
	unsigned int	hh_len;
	seqlock_t	hh_lock;
	/* cached hardware header; allow for machine alignment needs.        */
	unsigned long	hh_data[HH_DATA_ALIGN(LL_MAX_HEADER) / sizeof(long)];
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
  for each possible CPU
    struct softnet_data initialization
    sd->backlog.poll = process_backlog

  open_softirq(NET_TX_SOFTIRQ, net_tx_action)
  open_softirq(NET_RX_SOFTIRQ, net_rx_action)  

```

#### Key functions
* `netif_receive_skb`
`netif_receive_skb` should be called from device drivers for NAPI.
```
netif_receive_skb
  netif_receive_skb_internal
    __netif_receive_skb
      __netif_receive_skb_one_core
        __netif_receive_skb_core
```
More direct receive version of netif_receive_skb().  It should only be used by callers that have a need to skip RPS and Generic XDP.
Caller must also take care of handling if pfmemalloc.
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
    tcf_classify
      TC_ACT_OK: skb->tc_index = cl_res.classid
      TC_ACT_REDIRECT:  skb_do_redirect  ==>  __bpf_redirect  // BPF
  nf_ingress
    nf_hook_ingress

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
The handling routine of `NET_TX_SOFTIRQ` handles two main things:
  - The completion queue of the softnet_data structure for the executing CPU.
  - The output queue of the softnet_data structure for the executing CPU.
```
net_tx_action
  free skb in sd->completion_queue  // added in dev_kfree_skb_irq() by hardirq
  for each Qdisc in sd->output_queue  // the queue quota runs out or dev busy or queue stopped
    spin_lock(root_lock)
    qdisc_run     // process the queues

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
For IPv4, `dev_add_pack(&ip_packet_type)` is added in `inet_init` in `af_inet.c`
`ip_rcv`, the `struct packet_type.func` is called in the softirq.

```
static struct packet_type ip_packet_type __read_mostly = {
	.type = cpu_to_be16(ETH_P_IP),
	.func = ip_rcv,
	.list_func = ip_list_rcv,
};
```

* `netif_napi_add`
```
netif_napi_add
  init_gro_hash(napi)
  napi->poll = poll
  list_add(&napi->dev_list, &dev->napi_list)
  napi_hash_add(napi)
```

#### Packat processing path
* `netif_rx` in net/core/dev.c
Completes the interrupt handling. This function receives a packet from a device driver and queues it for the upper protocols to process.
```
netif_rx  ==>   netif_rx_internal
  #ifdef CONFIG_RPS
    cpu = get_rps_cpu
    enqueue_to_backlog
  #endif
    enqueue_to_backlog
      __skb_queue_tail(&sd->input_pkt_queue, skb) // skbs in input_pkt_queue will be processed as backlog
```

* `process_backlog`
```
process_backlog
  while skb = __skb_dequeue(&sd->process_queue)
    __netif_receive_skb(skb);

  skb_queue_splice_tail_init(&sd->input_pkt_queue, &sd->process_queue) // add the current input_pkt_queue to process_queue, which will be processed as backlog
```
The code is tricky, read the original code
```
enqueue_to_backlog
  if qlen <= netdev_max_backlog && !skb_flow_limit(skb, qlen)
    __skb_queue_tail(&sd->input_pkt_queue, skb)
    DONE

  if !__test_and_set_bit(NAPI_STATE_SCHED, &sd->backlog.state)
    if !rps_ipi_queued(sd)    // if the packet is on other cpus' queue, raise the softirq
      ____napi_schedule(sd, &sd->backlog) // NAPI to process current CPU's backlog
  }
  goto enqueue;
```

* `dev_direct_xmit`
```
dev_direct_xmit
  validate_xmit_skb_list

  skb_get_tx_queue

  netdev_start_xmit

  dev_xmit_complete
```


* `dev_queue_xmit`
Used by protocol instances of the higher protocols to send a packet in the form of the socket buffer over a network device
```
dev_queue_xmit  ==> __dev_queue_xmit
  sch_handle_egress
  txq = netdev_pick_tx
  q = txq->qdisc
  if q->enqueue
    __dev_xmit_skb

    job done!

    validate_xmit_skb
    /* The device has no queue. Common case for software devices:
  	 * loopback, all the sorts of tunnels...   
    dev_hard_start_xmit    
```

```
if (txq->xmit_lock_owner != cpu) {
  if (unlikely(__this_cpu_read(xmit_recursion) >
         XMIT_RECURSION_LIMIT))
    goto recursion_alert;
```
Background about the above code block:
There’s two cases: the transmit lock on this device queue is owned by this CPU or not. If so, a counter variable xmit_recursion, which is allocated per-CPU, is checked here to determine if the count is over the RECURSION_LIMIT. It is possible that one program could attempt to send data and get preempted right around this place in the code. Another program could be selected by the scheduler to run. If that second program attempts to send data as well and lands here. So, the xmit_recursion counter is used to prevent more than RECURSION_LIMIT programs from racing here to transmit data.

```
__dev_xmit_skb
  if TCQ_F_NOLOCK
    if __QDISC_STATE_DEACTIVATED // qdisc is deactived
      __qdisc_drop
    else
      q->enqueue
      qdisc_run(q)

  // Heuristic to force contended enqueues to serialize on a separate lock before trying to get qdisc main lock.
  if TCQ_F_CAN_BYPASS && !qdisc_qlen(q) && qdisc_run_begin(q)
    // TCQ_F_CAN_BYPASS: bypass the queuing system, no traffic shaping
    // !qdisc_qlen: no waiting packets in the queue
    // qdisc_run_begin: seqlock is hold or is_running return false, otherwise set to running
  if sch_direct_xmit // sec traffic-control.md
    __qdisc_run   
  qdisc_run_end  
  else
    q->enqueue(skb,...)
    if qdisc_run_begin
      __qdisc_run
```


```
netdev_pick_tx
  struct net_device_ops *ops = dev->netdev_ops
  if ops->ndo_select_queue)
      // The driver implements ndo_select_queue, choose a TX queue more intelligently in a hardware or specific way
			queue_index = ops->ndo_select_queue(dev, skb, sb_dev, __netdev_pick_tx);
	else
      // The driver does not implement `ndo_select_queue, so the kernel should pick
			queue_index = __netdev_pick_tx(dev, skb, sb_dev);
```

```
__netdev_pick_tx
   sk_tx_queue_get // cached in skb?

   if not cached or ooo_okay
      get_xps_queue   // Transmit Packet Steering.
      skb_tx_hash
      sk_tx_queue_set // if not cached and can cache it
```
If the `ooo_okay` flag is set. If this flag is set, this means that out of order packets are allowed now. The protocol layers must set this flag appropriately. The TCP protocol layer sets this flag when all outstanding packets for a flow have been acknowledged. When this happens, the kernel can choose a different TX queue for this packet.


```
skb_tx_hash
  if dev->num_tc  // device supports hardware based traffic control
    tc = netdev_get_prio_tc_map(dev, skb->priority)

  if skb_rx_queue_recorde // has been rcv or forwarded
      hash = skb_get_rx_queue(skb)
      while (unlikely(hash >= qcount))
        hash -= qcount;
      return hash + qoffset   // don't understand the logic behind it :(

  reciprocal_scale(skb_get_hash(skb), qcount) + qoffset

```

```
__netif_schedule ==>		__netif_reschedule
  *sd->output_queue_tailp = q
  sd->output_queue_tailp = &q->next_sched
  raise_softirq_irqoff(NET_TX_SOFTIRQ)  // at this moment, still in user context, raise softirq, and deferred for kthread to process the rest
```


* `dev_hard_start_xmit`
Call down to the device driver to actually do the transmit operation.
```
dev_hard_start_xmit
  for each skb in skb list
    xmit_one
      netdev_start_xmit
```

```
netdev_start_xmit
  struct net_device_ops *ops = dev->netdev_ops
   __netdev_start_xmit(ops, ...)
      ops->ndo_start_xmit
```

```
validate_xmit_skb_list
  for each skb in skb list
    validate_xmit_skb
```

```
validate_xmit_skb
  validate_xmit_vlan
    __vlan_hwaccel_push_inside
      vlan_insert_tag_set_proto(skb, skb->vlan_proto, skb_vlan_tag_get(skb))
      __vlan_hwaccel_clear_tag

  sk_validate_xmit_skb
    sk->sk_validate_xmit_skb

  if netif_needs_gso
    skb_gso_segment
      skb_mac_gso_segment
         ptype->callbacks.gso_segment

  if skb_needs_linearize
  /* If packet is not checksummed and device does not support checksumming for this protocol, complete checksumming here.
  skb_set_inner_transport_header  or   skb_set_transport_header

  validate_xmit_xfrm
```


* `dev_open`

* `dev_close`


#### GRO

A list of GRO offload filters is traversed to allow the higher level protocol stacks to act on a piece of data which is being considered for GRO. This is done so that the protocol layers can let the network device layer know if this packet is part of a network flow that is currently being receive offloaded and handle anything protocol specific that should happen for GRO.
```
napi_gro_receive
  skb_gro_reset_offset // initialization of the skb's cb NAPI_GRO_CB(skb)
  dev_gro_receive(napi, skb)
  napi_skb_finish   // memory clean up for different return cases
```

```
dev_gro_receive
  struct list_head *head = &offload_base;
  list_for_each_entry_rcu(ptype, head, list)
  pp = INDIRECT_CALL_INET(ptype->callbacks.gro_receive, ipv6_gro_receive, inet_gro_receive, gro_head, skb)

  if napi->gro_hash[hash].count >= MAX_GRO_SKBS
		gro_flush_oldest(gro_head)

  list_add(&skb->list, gro_head);
  ret = GRO_HELD;

  gro_pull_from_frag0(skb, grow);

```

```
napi_gro_complete
  *head = &offload_base
  list_for_each_entry_rcu(ptype, head, list)
  INDIRECT_CALL_INET(ptype->callbacks.gro_complete, ipv6_gro_complete, inet_gro_complete, skb, 0)
  netif_receive_skb_internal
```

### RPS
```
get_rps_cpu
```

```
rps_ipi_queued
  if sd != mysd
    sd->rps_ipi_next = mysd->rps_ipi_list
    mysd->rps_ipi_list = sd

    __raise_softirq_irqoff(NET_RX_SOFTIRQ)
```

#### XSP
* `get_xps_queue`
```
get_xps_queue
  __get_xps_queue_idx
    if dev->num_tc
      tci *= dev->num_tc
      tci += netdev_get_prio_tc_map(dev, skb->priority)

    map = dev_maps->attr_map[tci]
    map->queues[reciprocal_scale(skb_get_hash(skb), map->len)]
      // reciprocal_scale scale" a value into range [0, @ep_ro)
```

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
  netdev_start_xmit

```

```
netif_receive_generic_xdp
  set xdp->data to the MAC addr, xdp->end to the end of header (before the package)
  bpf_prog_run_xdp
  handle bpf_xdp_adjust_head and bpf_xdp_adjust_tail, adjust skb and skb->mac_header
```
