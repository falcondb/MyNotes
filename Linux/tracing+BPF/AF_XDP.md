## AF_XDP
### In userland
#### AF_XDP or XDP socket key data structures

```
# tools/lib/bpf/xsk.c

struct xsk_umem {
	struct xsk_ring_prod *fill;
	struct xsk_ring_cons *comp;
	char *umem_area;
	struct xsk_umem_config config;
	int fd;
	int refcount;
};

struct xsk_umem_config {
	__u32 fill_size;
	__u32 comp_size;
	__u32 frame_size;
	__u32 frame_headroom;
	__u32 flags;
};

/* Do not access these members directly. Use the functions below. */
#define DEFINE_XSK_RING(name) \
struct name { \
	__u32 cached_prod; \
	__u32 cached_cons; \
	__u32 mask; \
	__u32 size; \
	__u32 *producer; \
	__u32 *consumer; \
	void *ring; \
	__u32 *flags; \
}
DEFINE_XSK_RING(xsk_ring_prod);
DEFINE_XSK_RING(xsk_ring_cons);

struct xsk_socket {
	struct xsk_ring_cons *rx;
	struct xsk_ring_prod *tx;
	__u64 outstanding_tx;
	struct xsk_umem *umem;
	struct xsk_socket_config config;
	int fd;
	int ifindex;
	int prog_fd;
	int xsks_map_fd;
	__u32 queue_id;
	char ifname[IFNAMSIZ];
};

struct xsk_socket_config {
	__u32 rx_size;
	__u32 tx_size;
	__u32 libbpf_flags;
	__u32 xdp_flags;
	__u16 bind_flags;
};

# uapi/linux/if_xdp.h
struct xdp_umem_reg {
	__u64 addr; /* Start of packet data area */
	__u64 len; /* Length of packet data area */
	__u32 chunk_size;
	__u32 headroom;
	__u32 flags;
};

struct xdp_mmap_offsets {
	struct xdp_ring_offset rx;
	struct xdp_ring_offset tx;
	struct xdp_ring_offset fr; /* Fill */
	struct xdp_ring_offset cr; /* Completion */
};

struct xdp_ring_offset {
	__u64 producer;
	__u64 consumer;
	__u64 desc;
	__u64 flags;
};

struct sockaddr_xdp {
	__u16 sxdp_family;
	__u16 sxdp_flags;
	__u32 sxdp_ifindex;
	__u32 sxdp_queue_id;
	__u32 sxdp_shared_umem_fd;
};

```

#### AF_XDP or XDP socket key execution flow

```
xsk_umem__create ==>
  umem->fd = socket(AF_XDP, SOCK_RAW, 0);
  xdp_umem_reg setup
  setsockopt(umem->fd, SOL_XDP, XDP_UMEM_REG, ...)
  setsockopt(umem->fd, SOL_XDP, XDP_UMEM_FILL_RING, ...)
  setsockopt(umem->fd, SOL_XDP, XDP_UMEM_COMPLETION_RING, ...)

  xsk_get_mmap_offsets ==> getsockopt(fd, SOL_XDP, XDP_MMAP_OFFSETS, ...)

  map = mmap(NULL, off.fr.desc + umem->config.fill_size * sizeof(__u64),
		   PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE, umem->fd, XDP_UMEM_PGOFF_FILL_RING);
  fill->producer = map + off.fr.producer;
  fill->consumer = map + off.fr.consumer;
  fill->ring = map + off.fr.desc;       
  fill->cached_cons = umem->config.fill_size;

  map = mmap(NULL, off.cr.desc + umem->config.comp_size * sizeof(__u64),
     PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE, umem->fd, XDP_UMEM_PGOFF_COMPLETION_RING);
  comp->producer = map + off.cr.producer;
 	comp->consumer = map + off.cr.consumer;
 	comp->ring = map + off.cr.desc;


xsk_socket__create ==>
    struct xsk_socket setup
    setsockopt(xsk->fd, SOL_XDP, XDP_RX_RING, ...)
    setsockopt(xsk->fd, SOL_XDP, XDP_TX_RING, ...)
    xsk_get_mmap_offsets();
    rx_map = mmap(NULL, off.rx.desc + xsk->config.rx_size * sizeof(struct xdp_desc),
        PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE, xsk->fd, XDP_PGOFF_RX_RING);
        rx->producer = rx_map + off.rx.producer;
		rx->consumer = rx_map + off.rx.consumer;
		rx->ring = rx_map + off.rx.desc;

    tx_map = mmap(NULL, off.tx.desc + xsk->config.tx_size * sizeof(struct xdp_desc),
        PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE, xsk->fd, XDP_PGOFF_TX_RING);
    tx->producer = tx_map + off.tx.producer;
		tx->consumer = tx_map + off.tx.consumer;
		tx->ring = tx_map + off.tx.desc;

    struct sockaddr_xdp (sxdp) set up
    bind(xsk->fd, (struct sockaddr *)&sxdp, sizeof(sxdp));
    xsk_setup_xdp_prog ==>
      bpf_get_link_xdp_id
      if no prog
        xsk_create_bpf_maps ==>
          bpf_create_map_name(BPF_MAP_TYPE_XSKMAP, "xsks_map", ..., max_queues, 0);
        xsk_load_xdp_prog ==>
          BPF byte code of calling bpf_redirect_map()
          bpf_load_program
          bpf_set_link_xdp_fd
      else prog exists
        xsk->prog_fd = bpf_prog_get_fd_by_id
        xsk_lookup_bpf_maps
          bpf_obj_get_info_by_fd to get number of maps in prog_info.nr_map_ids
          allocate memory for prog_info.map_ids
          bpf_obj_get_info_by_fd again to get map ids
          for each map id
            bpf_map_get_fd_by_id
            bpf_obj_get_info_by_fd
            if (!strcmp(map_info.name, "xsks_map"))
              xsk->xsks_map_fd = fd;
      if xsk->rx
  		  xsk_set_bpf_maps


```

### In kernel
#### AF_XDP or XDP socket key data structures

```
static const struct net_proto_family xsk_family_ops = {
	.family = PF_XDP,
	.create = xsk_create,
	.owner	= THIS_MODULE,
};

static const struct proto_ops xsk_proto_ops = {
	.family		= PF_XDP,
	.owner		= THIS_MODULE,
	.release	= xsk_release,
	.bind  		= xsk_bind,
	.connect	= sock_no_connect,
	.socketpair	= sock_no_socketpair,
	.accept		= sock_no_accept,
	.getname	= sock_no_getname,
	.poll  		= xsk_poll,
	.ioctl		= sock_no_ioctl,
	.listen		= sock_no_listen,
	.shutdown	= sock_no_shutdown,
	.setsockopt	= xsk_setsockopt,
	.getsockopt	= xsk_getsockopt,
	.sendmsg	= xsk_sendmsg,
	.recvmsg	= sock_no_recvmsg,
	.mmap		  = xsk_mmap,
	.sendpage	= sock_no_sendpage,
};

static struct pernet_operations xsk_net_ops = {
	.init = xsk_net_init,
	.exit = xsk_net_exit,
};

```
#### AF_XDP or XDP socket key execution flow

```
fs_initcall(xsk_init) == asm(".section	\"" #__sec ".init\", ...) or call the fn for module
xsk_init

```


```
xsk_create
  sk_alloc for struct net
  sock->ops = &xsk_proto_ops
  sk_add_node_rcu(sk, &net->xdp.list)
    refcount_inc(&sk->sk_refcnt);
    hlist_add_head_rcu(&sk->sk_node, list)
  local_bh_disable
    sock_prot_inuse_add ==> net->core.prot_inuse->val[prot->inuse_idx] + 1
  local_bh_enable

```

```
# net/xdp/xsk.c
__init xsk_init
  proto_register(&xsk_proto, 0 /* no slab */)
    slab path: TOBESTUDIED
    mutex_lock(&proto_list_mutex);
    assign_proto_idx ==> set_bit(prot->inuse_idx)
    list_add(&prot->node, &proto_list);
    mutex_unlock

  sock_register(&xsk_family_ops)
    spin_lock(&net_family_lock);
    rcu_assign_pointer(net_families[ops->family], ops);
    spin_unlock(&net_family_lock);

  register_pernet_subsys(&xsk_net_ops) ==> __register_pernet_operations # net/core/net_namespace.c
    list_add_tail(&ops->list, list)
    ops_init; ist_add_tail(&net->exit_list, &net_exit_list);

  register_netdevice_notifier(&xsk_netdev_notifier)
    raw_notifier_chain_register
    list_for_each_entrynetVAR, &net_namespace_list, list)
      call_netdevice_register_net_notifiers(nb, net)

xsk_setsockopt
  ?????
```
