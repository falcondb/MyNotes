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
# for the registration parameters
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

/* Rx/Tx descriptor */
struct xdp_desc {
	__u64 addr;
	__u32 len;
	__u32 options;
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
In userland
```
// tools/lib/bpf/xsk.c
xsk_umem__create (..., void *umem_area, ...) ==>
  umem->fd = socket(AF_XDP, SOCK_RAW, 0);
  xdp_umem_reg setup
  setsockopt(umem->fd, SOL_XDP, XDP_UMEM_REG, ...)
  setsockopt(umem->fd, SOL_XDP, XDP_UMEM_FILL_RING, ...)
  setsockopt(umem->fd, SOL_XDP, XDP_UMEM_COMPLETION_RING, ...)

	struct xdp_umem_reg mr->addr  = umem_area // pass the UMEM addr in userland to kernel
  xsk_get_mmap_offsets ==> getsockopt(fd, SOL_XDP, XDP_MMAP_OFFSETS, mr)

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
		umem->refcount++ > 0 ? xsk->fd = socket(AF_XDP, SOCK_RAW, 0) : xsk->fd = umem->fd // umem != in use, xsk.fd = umem.fd
    setsockopt(xsk->fd, SOL_XDP, XDP_RX_RING, ...)
    setsockopt(xsk->fd, SOL_XDP, XDP_TX_RING, ...)
    xsk_get_mmap_offsets();  // the members' offset in ring struct

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

```
# xdp/xdp_sock.h

struct xdp_sock {
	struct sock sk;
	struct xsk_queue *rx;
	struct net_device *dev;
	struct xdp_umem *umem;
	struct list_head flush_node;
	u16 queue_id;
	bool zc;
	enum { XSK_READY = 0, XSK_BOUND, XSK_UNBOUND,} state;
	struct mutex mutex;
	struct xsk_queue *tx;
	struct list_head list;
	spinlock_t tx_completion_lock;
	spinlock_t rx_lock;
	u64 rx_dropped;
	struct list_head map_list;
	spinlock_t map_list_lock;
};

struct xdp_umem {
	struct xsk_queue *fq;
	struct xsk_queue *cq;
	struct xdp_umem_page *pages;
	u64 chunk_mask;
	u64 size;
	u32 headroom;
	u32 chunk_size_nohr;
	struct user_struct *user;
	unsigned long address;
	refcount_t users;
	struct work_struct work;
	struct page **pgs;
	u32 npgs;
	u16 queue_id;
	u8 need_wakeup;
	u8 flags;
	int id;
	struct net_device *dev;
	struct xdp_umem_fq_reuse *fq_reuse;
	bool zc;
	spinlock_t xsk_list_lock;
	struct list_head xsk_list;
};

struct xsk_queue {
	u64 chunk_mask;
	u64 size;
	u32 ring_mask;
	u32 nentries;
	u32 prod_head;
	u32 prod_tail;
	u32 cons_head;
	u32 cons_tail;
	struct xdp_ring *ring;
	u64 invalid_descs;
};

struct xdp_ring {
	u32 producer;
	u32 consumer;
	u32 flags;
};

struct xsk_map {
	struct bpf_map map;
	struct list_head __percpu *flush_list;
	spinlock_t lock; /* Synchronize map updates */
	struct xdp_sock *xsk_map[];
};

struct xsk_map_node {
	struct list_head node;
	struct xsk_map *map;
	struct xdp_sock **map_entry;
};

struct xdp_umem_page {
	void *addr;
	dma_addr_t dma;
};

```

#### AF_XDP or XDP socket key execution flow

```
fs_initcall(xsk_init) == asm(".section	\"" #__sec ".init\", ...)
```


```
xsk_family_ops = { .create = xsk_create };

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
  case XDP_RX_RING:
  case XDP_TX_RING:
    mutex_lock(&xs->mutex);
    xsk_init_queue(entries, xs->tx/rx, false);
      xskq_create
        kzalloc(struct xsk_queue q) // slab and zero
        q->ring = __get_free_pages(GFP_KERNEL | __GFP_ZERO | __GFP_NOWARN |
		    __GFP_COMP  | __GFP_NORETRY, get_order(size));
    mutex_unlock(&xs->mutex);
  case XDP_UMEM_REG:
    mutex_lock(&xs->mutex);
    xdp_umem_create
      ida_simple_get  // get an id from idr/xarray
      xdp_umem_reg
        xdp_umem_account_pages  // check the locked-maps of the user with rlimit MEMLOCK
        xdp_umem_pin_pages
          umem->pgs = kcalloc(umem->npgs, sizeof(*umem->pgs), GFP_KERNEL | __GFP_NOWARN); // struct page
          down_read(&current->mm->mmap_sem);
          get_user_pages(umem->address, ..., FOLL_WRITE | FOLL_LONGTERM,...) // pin_user_pages.rst.
          up_read(&current->mm->mmap_sem)
        umem->pages = kcalloc(umem->npgs, sizeof(*umem->pages), GFP_KERNEL); // struct xdp_umem_page
        xdp_umem_map_pages
    mutex_unlock(&xs->mutex);
  case XDP_UMEM_FILL_RING:
  case XDP_UMEM_COMPLETION_RING:  
    xsk_init_queue(entries, &xs->umem->fq/cq, true);
      xskq_create
        ...


xsk_mmap
  offset = (loff_t)vma->vm_pgoff << PAGE_SHIFT;
  if (offset == XDP_PGOFF_RX_RING)
    q = READ_ONCE(xs->rx);
  else if (offset == XDP_PGOFF_TX_RING)
    q = READ_ONCE(xs->tx);
  else if offset == XDP_UMEM_PGOFF_FILL_RING
    q = READ_ONCE(umem->fq);
  else if (offset == XDP_UMEM_PGOFF_COMPLETION_RING)
		q = READ_ONCE(umem->cq);

  pfn = virt_to_phys(q->ring) >> PAGE_SHIFT
  remap_pfn_range(vma, vma->vm_start, pfn, ...) //remap kernel memory to userspace, see memory.md
```
