## AF_XDP

#### AF_XDP or XDP socket key data structures
In userland:
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
In userland
```
xsk_umem__create ==>
  umem->fd = socket(AF_XDP, SOCK_RAW, 0);
  xdp_umem_reg setup
  setsockopt(umem->fd, SOL_XDP, XDP_UMEM_FILL_RING, ...)
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
