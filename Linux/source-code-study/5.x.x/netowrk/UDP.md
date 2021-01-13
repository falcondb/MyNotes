## UDP

### egress

* `udp_send_skb`
```
udp_send_skb
  // UDP cork
  // UDP checksum
  ip_send_skb
```

### UDP corking
UDP corking is a feature that allows a user program request that the kernel accumulate data from multiple calls to send into a single datagram before sending. There are two ways to enable this option in your user program:


* `ip_append_data`
Used by UDP, ICMP & RAW.
`ip_append_data()` and `ip_append_page()` can make one large IP datagram from many pieces of data
```
ip_append_data
	if skb_queue_empty(&sk->sk_write_queue)
		ip_setup_cork(sk, &inet->cork.base, ...)	// see UPD.md

	__ip_append_data

```
