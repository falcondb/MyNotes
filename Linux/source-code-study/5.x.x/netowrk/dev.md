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
	unsigned int		processed, time_squeeze, received_rps;
	struct sd_flow_limit __rcu *flow_limit;
	struct Qdisc		*output_queue;
	struct Qdisc		**output_queue_tailp;
	struct sk_buff		*completion_queue;
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

#### net/dev.c
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
* `netif_receive_skb_core`
```
netif_receive_skb_core
  __netif_receive_skb_one_core
    __netif_receive_skb_core

```
* `net_tx_action`
the handling routine of `NET_RX_SOFTIRQ`
```
net_tx_action

```
* `net_rx_action`
```
net_rx_action
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

* `dev_queue_xmit`
Used by protocol instances of the higher protocols to send a packet in the form of the socket buffer over a network device

* `dev_open`

* `dev_close`
