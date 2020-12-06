## TCP

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
