## TCP
TCP is just too complex to me. Need more time to consume it.

### TCP output
[David Milller's blog on TCP output](http://vger.kernel.org/~davem/tcp_output.html)
```
# net/ipv4/tcp_output.c
tcp_mtu_probe
tcp_write_xmit
__tcp_retransmit_skb
tcp_send_synack
__tcp_send_ack
tcp_send_syn_data
tcp_xmit_probe_skb
tcp_write_wakeup
all of above call tcp_transmit_skb()

tcp_transmit_skb ==> __tcp_transmit_skb
  skb_push(skb, tcp_header_size)
  skb->destructor = __sock_wfree / sock_wfree
  /* Build TCP header and checksum it. */
  th = (struct tcphdr *)skb->data;
  th.XXX = ZZZ
  icsk->icsk_af_ops->send_check
  icsk->icsk_af_ops->queue_xmit
```

### TCP control block
[David's blog on TCP CB](http://vger.kernel.org/~davem/tcp_skbcb.html)

```
# net/tcp.h
struct tcp_skb_cb {
  __u32		seq;		/* Starting sequence number	*/
	__u32		end_seq;	/* SEQ + FIN + SYN + datalen	*/
  __u32		ack_seq;	/* Sequence number ACK'd	*/
  struct inet_skb_parm	h4;
  struct {
    __u32 flags;
    struct sock *sk_redir;
    void *data_end;
  } bpf;
  ...
}
```


### ipv4/tcp_ipv4.c

```
struct proto tcp_prot = {
	.name			= "TCP",
	.owner			= THIS_MODULE,
	.close			= tcp_close,
	.pre_connect		= tcp_v4_pre_connect,
	.connect		= tcp_v4_connect,
	.disconnect		= tcp_disconnect,
	.accept			= inet_csk_accept,
	.ioctl			= tcp_ioctl,
	.init			= tcp_v4_init_sock,
	.destroy		= tcp_v4_destroy_sock,
	.shutdown		= tcp_shutdown,
	.setsockopt		= tcp_setsockopt,
	.getsockopt		= tcp_getsockopt,
	.keepalive		= tcp_set_keepalive,
	.recvmsg		= tcp_recvmsg,
	.sendmsg		= tcp_sendmsg,
	.sendpage		= tcp_sendpage,
	.backlog_rcv		= tcp_v4_do_rcv,
	.release_cb		= tcp_release_cb,
	.hash			= inet_hash,
	.unhash			= inet_unhash,
	.get_port		= inet_csk_get_port,
	.enter_memory_pressure	= tcp_enter_memory_pressure,
	.leave_memory_pressure	= tcp_leave_memory_pressure,
	.stream_memory_free	= tcp_stream_memory_free,
	.sockets_allocated	= &tcp_sockets_allocated,
	.orphan_count		= &tcp_orphan_count,
	.memory_allocated	= &tcp_memory_allocated,
	.memory_pressure	= &tcp_memory_pressure,
	.sysctl_mem		= sysctl_tcp_mem,
	.sysctl_wmem_offset	= offsetof(struct net, ipv4.sysctl_tcp_wmem),
	.sysctl_rmem_offset	= offsetof(struct net, ipv4.sysctl_tcp_rmem),
	.max_header		= MAX_TCP_HEADER,
	.obj_size		= sizeof(struct tcp_sock),
	.slab_flags		= SLAB_TYPESAFE_BY_RCU,
	.twsk_prot		= &tcp_timewait_sock_ops,
	.rsk_prot		= &tcp_request_sock_ops,
	.h.hashinfo		= &tcp_hashinfo,
	.no_autobind		= true,
	.compat_setsockopt	= compat_tcp_setsockopt,
	.compat_getsockopt	= compat_tcp_getsockopt,
	.diag_destroy		= tcp_abort,
};
```

* `tcp_v4_init_sock`
```
tcp_v4_init_sock
	tcp_init_sock  // initial values
	icsk->icsk_af_ops = &ipv4_specific;

```

```
const struct inet_connection_sock_af_ops ipv4_specific = {
	.queue_xmit	   = ip_queue_xmit,
	.send_check	   = tcp_v4_send_check,  // update checksum in tcp header
	.rebuild_header	   = inet_sk_rebuild_header,
	.sk_rx_dst_set	   = inet_sk_rx_dst_set,
	.conn_request	   = tcp_v4_conn_request,
	.syn_recv_sock	   = tcp_v4_syn_recv_sock,
	.net_header_len	   = sizeof(struct iphdr),
	.setsockopt	   = ip_setsockopt,
	.getsockopt	   = ip_getsockopt,
	.addr2sockaddr	   = inet_csk_addr2sockaddr,
	.sockaddr_len	   = sizeof(struct sockaddr_in),
	.compat_setsockopt = compat_ip_setsockopt,
	.compat_getsockopt = compat_ip_getsockopt,
	.mtu_reduced	   = tcp_v4_mtu_reduced,
};

```  


* `tcp_v4_connect`
```
tcp_v4_connect
  ip_route_connect
    ip_route_connect_init
      flowi4_init_output
    __ip_route_output_key
    flowi4_update_output
    ip_route_output_flow
      __ip_route_output_key => ip_route_output_key_hash => ip_route_output_key_hash_rcu
        if fl4->flowi4_oif == 0
          dev_out = __ip_dev_find
          fl4->flowi4_oif = dev_out->ifindex
        if fl4->flowi4_oif
          dev_get_by_index_rcu
        fib_lookup  ==> fib_table_lookup  // LC-trie fib_trie.txt
        fib_select_path
        __mkroute_output
  inet_hash_connect
  sk_set_txhash
  ip_route_newports
  tcp_connect   //Build a SYN and send it off
    tcp_connect_init
      ...
      tcp_rwnd_init_bpf ==> tcp_call_bpf  // receive window
        See TCP common section below
      tcp_select_initial_window
    sk_stream_alloc_skb   // net/ipv4/tpc.c
      if !size   ==>  sk->sk_tx_skb_cache
      alloc_skb_fclone  ==>  __alloc_skb(size, priority, SKB_ALLOC_FCLONE, NUMA_NO_NODE)  //skbuff.c
        see skbuff-execution.md
      reset the 4 skb offsets
    tcp_connect_queue_skb  // link struct sock and struct sk_buff data
    tcp_ecn_send_syn       // Packet ECN state for a SYN
    tcp_rbtree_insert      // tpc_input.c Insert skb into rb tree, ordered by TCP_SKB_CB(skb)->seq
    tcp_transmit_skb       // tcp_transmit_skb section
    tcp_send_head
```

* `tcp_shutdown`
```
tcp_shutdown
  tcp_set_state
  // the close state map
  if BPF_SOCK_OPS_STATE_CB_FLAG
    call tcp_call_bpf with BPF_SOCK_OPS_STATE_CB
```


* `inet_csk_accept`
```
# net/ipv4/inet_connection_sock.c
inet_csk_accept
  req = reqsk_queue_remove(icsk->icsk_accept_queue, sock)
  return req->sk
```

* `tcp_recvmsg`
```
# net/ipv4/tcp.c
tcp_recvmsg   // copies from a sock struct into the user buffer
  SIGURG out-of-band data
  skb_queue_walk sk->sk_receive_queue
    offset = *seq - TCP_SKB_CB(skb)->seq
    if offset < skb->len    goto found_ok_skb   // package with the next expected seq?

    skb_copy_datagram_msg  ==>  skb_copy_datagram_iter //skbuff.h
      __skb_datagram_iter with func simple_copy_to_iter
        simple_copy_to_iter ==> copy_to_iter // uio.h ==>  _copy_to_iter  // lib/iov_iter.c
          // TOBESTUDIED the MACRO

    tcp_rcv_space_adjust  // tpc_input.c calculates the appropriate TCP receive buffer space
      // the receive buffer size recalcation.
    release_sock    //sock.c
      __release_sock
        for skb in sock backlog
          sk_backlog_rcv  // sk->sk_prot->backlog_rcv = tcp_v4_do_rcv for ipv4 in tcp_ipv4.c
            See tcp_v4_do_rcv section

```

* `tcp_sendmsg`
```
# tpc.c
tcp_sendmsg
  lock_sock  with  sk->sk_lock.wq
  tcp_sendmsg_locked
    sk_stream_wait_connect
    more data left after segment
      tcp_write_queue_tail    // sk->sk_write_queue's skb_head->next
      tcp_rtx_and_write_queues_empty
      sk_stream_alloc_skb
        alloc_skb_fclone   ==>   __alloc_skb
        skb_reserve(sk_prot->max_header)
      skb_entail
        tcp_add_write_queue_tail
        tcp_slow_start_after_idle_check
      tcp_push
        __tcp_push_pending_frames
          tcp_write_xmit
```


* `tcp_v4_do_rcv`
```
# tcp_ipv4.c
tcp_v4_do_rcv
  if sk->sk_state == TCP_ESTABLISHED
    tcp_rcv_established
      // the fast path when recved seq is the expect and the previous is acked
        tcp_ack
          tcp_ack_update_window            
          tcp_in_ack_event
            inet_connection_sock->icsk_ca_ops(struct tcp_congestion_ops)->in_ack_event(struct tcp_congestion_ops)
            // seems in_ack_event is not set for IPv4
          tcp_rack_update_reo_wnd // RFC 3708
        __kfree_skb
        tcp_data_snd_check
      // slow path

  tcp_rcv_state_process
```
Function header of `tcp_rcv_established`, the fast path of TCP receive
> TCP receive function for the ESTABLISHED state.
It is split into a fast path and a slow path. The fast path is
disabled when:
	- A zero window was announced from us - zero window probing is only handled properly in the slow path.
  - Out of order segments arrived.
 	- Urgent data is expected.
 	- There is no buffer space left
 	- Unexpected TCP flags/window values/header lengths are received
 	- Data is sent in both directions. Fast path only supports pure senders or pure receivers (this means either the sequence number or the ack value must stay constant)
 	- Unexpected TCP option.


* `tcp_v4_rcv`


### TCP common
* `alloc_skb`
```
# skbuff.c
  see skbuff-execution.md
```

* `tcp_call_bpf`
```
BPF_CGROUP_RUN_PROG_SOCK_OPS    //bpf-cgroup.h
  __cgroup_bpf_run_filter_sock_ops  // bpf/cgroup.c
    sock_cgroup_ptr   // cgroup.h
      struct sock_cgroup_data.val // cgroup pointer in a combination of net_cls & net_prio & cgroup
    BPF_PROG_RUN_ARRAY  ==> __BPF_PROG_RUN_ARRAY(array, ctx, func, false)
      see bpf/key-execution-flow.md
```


* `__inet_lookup_skb`

```
__inet_lookup_skb  ==>  __inet_lookup    //inet_hashtables.h
    __inet_lookup_established or  __inet_lookup_listener    //inet_hashtables.c
      inet_ehashfn(net, daddr, hnum, saddr, sport)    ==>   __inet_ehashfn
          jhash_3words    // Jenkins hash function
      INET_MATCH(sk, net, acookie, saddr, daddr, ports, dif, sdif))
```
