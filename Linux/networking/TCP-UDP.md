[A Map of the Networking Code in Linux Kernel 2.4.20](http://www.martin-flatin.org/papers/tr-datatag-2004-1.pdf)


## The segment offsets


- copied_seq: Head of yet unread data
- rcv_wup: rcv_nxt on last window update sent
- rcv_nxt: What we want to receive next
- rcv_wnd: current receiver window

- snd_nxt: ext sequence we send
- snd_una:  First byte we want an ack for
- snd_sml:  Last byte of the most recently transmitted small packet
- snd_wl1:  Sequence for window update
- snd_wnd:  The window we expect to receive
- max_window: Maximal window ever seen from peer
- snd_up:   Urgent pointer

- tcp_skb_cb
  - seq: Starting sequence number
  - end_seq: SEQ + FIN + SYN + datalen
  - ack_seq: Sequence number ACK'd


### tcp ingress data processing

when the TCP connection is established
`tcp_rcv_established`
  if the seg is the next we are expecting, go through the fastpath
    if empty payload, just `tcp_ack` incoming ack handler, `tcp_data_snd_check`
      `tcp_push_pending_frames`
        `tcp_write_xmit` see ingree&egress.md
    else a payload here,
      `tcp_queue_rcv`
        `tcp_try_coalesce`
        `__skb_queue_tail(sk->sk_receive_queue, skb)` The sk_receive_queue will be consumed by tcp_cleanup_rbuf

  otherwise, go through the slowpath, do different checks, then
    `tcp_data_queue`
      handle out-of-window
      `tcp_queue_rcv`
      handle out-of-order queue
      `tcp_data_queue_ofo` add to out-of-order queue
    eventually `__tcp_ack_snd_check`


`tcp_ack`
  `tcp_clean_rtx_queue`


### tcp egress data processing
`tcp_write_wakeup` \ `tcp_write_xmit`
  `tcp_transmit_skb`
    `icsk_af_ops->queue_xmit`
  `tcp_event_new_data_sent`
    unlink skb from sk_write_queue
    insert skb to rbtree tcp_rtx_queue



### hook points for network stack walking
tcp_sendmsg struct sock, msghdr

ip_local_out sock sk_buff // enter the network layer

ip_output sock sk_buff ==> ip_finish_output sock sk_buff
ip_mc_output sock sk_buff ==> ip_mc_finish_output sock sk_buff

ip_finish_output2 sock sk_buff ==> neigh_output neighbour sk_buff ==> dev_queue_xmit sk_buff // neighbouring

// TODO: local delievery

// device TC
__dev_queue_xmit / trace_net_dev_queue sk_buff
  __dev_xmit_skb sk_buff Qdisc, net_device netdev_queue // with queue
    __qdisc_run
      __netif_reschedule Qdisc
        NET_TX_SOFTIRQ net_tx_action softirq_action

  dev_hard_start_xmit sk_buff net_device netdev_queue // without queue
    xmit_one sk_buff, net_device netdev_queue
      trace_net_dev_start_xmit
      netdev_start_xmit
        net_device_ops->ndo_start_xmit
      trace_net_dev_xmit
