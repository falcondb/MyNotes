## XDP in Kernel

### include/linux/net_device.h
```
struct netdev_bpf {
	enum bpf_netdev_command command;
	union {
		/* XDP_SETUP_PROG */
		struct {
			u32 flags;
			struct bpf_prog *prog;
			struct netlink_ext_ack *extack;
		};
		/* XDP_QUERY_PROG, XDP_QUERY_PROG_HW */
		struct {
			u32 prog_id;
			u32 prog_flags;  /* flags with which program was installed */
		};
		/* BPF_OFFLOAD_MAP_ALLOC, BPF_OFFLOAD_MAP_FREE */
		struct {
			struct bpf_offloaded_map *offmap;
		};
		/* XDP_SETUP_XSK_UMEM */
		struct {
			struct xdp_umem *umem;
			u16 queue_id;
		} xsk;
	};
};
```

### net/core/dev.c

```
__init rtnetlink_init
rtnl_register(PF_UNSPEC, RTM_NEWLINK, rtnl_newlink
rtnl_newlink
  __rtnl_newlink
    do_setlink
      dev_change_xdp_fd
        bpf_prog_get_type_dev(fd, BPF_PROG_TYPE_XDP, )
        dev_xdp_install
          bpf_op = generic_xdp_install
                   rcu_assign_pointer(dev->xdp_prog, new)

            int (*bpf_op_t)(struct net_device *dev, struct netdev_bpf *bpf)  // net_device.h

```

```
netif_receive_skb_core  ||  netif_receive_skb ==> netif_receive_skb_internal ==>  __netif_receive_skb
  __netif_receive_skb_one_core   ||  __netif_receive_skb_list_core
    __netif_receive_skb_core
      do_xdp_generic
        netif_receive_generic_xdp
          bpf_prog_run_xdp
            (*(prog)->bpf_func)(ctx, (prog)->insnsi)
              // bpf/core.c BPF JIT, TOBESTUDIED
        XDP_REDIRECT ==> xdp_do_generic_redirect
        XDP_TX       ==> generic_xdp_tx

```
