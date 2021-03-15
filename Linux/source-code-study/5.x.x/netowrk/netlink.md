## Netlink

[Netlink and usespace, netlink example](https://blog.csdn.net/qq_18144747/article/details/98179350)

### Key functions

* `af_netlink.c`

```
netlink_attachskb
  // Error handling
  netlink_skb_set_owner_r
    skb->sk = sk
```

```
__netlink_kernel_create
  sock_create_lite(PF_NETLINK, SOCK_DGRAM, unit, &sock)
    struct socket *sock = sock_alloc
  __netlink_create
      sock->ops = &netlink_ops
      struct sock *sk = sk_alloc(net, PF_NETLINK, GFP_KERNEL, &netlink_proto, 1)
      sock_init_data
      sk->sk_data_ready = netlink_data_ready
      nlk_sk(sk)->netlink_rcv = cfg->input
      netlink_insert(sk, 0) // insert to nl_table[0]
```

```
netlink_unicast
  if netlink_is_kernel    ==>   netlink_unicast_kernel

  netlink_attachskb
  netlink_sendskb
```

```
netlink_sendskb ==> __netlink_sendskb


```

```
netlink_rcv_skb
  cb    // the passed in callback

  netlink_ack if NLM_F_ACK

```
