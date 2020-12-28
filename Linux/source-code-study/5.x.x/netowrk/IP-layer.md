## IP layer

### net
* `netif_rx`

### IPv4
See `struct packet_type` definition in dev.md
```
static struct packet_type ip_packet_type __read_mostly = {
	.type = cpu_to_be16(ETH_P_IP),
	.func = ip_rcv,
	.list_func = ip_list_rcv,
};
```

### key functions

#### IPv4 initialization
* `net/ipv4/af_inet.c`
```
__init inet_init

```


#### igress
##### `ip_input.c`
* `ip_rcv`

* `ip_rcv_finsih`

* `ip_local_deliver`

* `ip_defrag`

##### `ipmr.c`
* `ip_mr_input`


#### ip_forward
* `ip_forward`

* `ip_forward_finish`

* ` ip_route_input` in `net/route.h`

* `ip_queue_xmit`

#### egress

* `__ip_queue_xmit` in `ipv4/ip_output.c`
```
ip_queue_xmit  ==>  __ip_queue_xmit
```

* `p_queue_xmit2`?

* `ip_output`

* `ip_finish_output`

* `ip_fragment`

* `dev_queue_xmit`


#### IP options
##### Key data structures
* `inet_sock.h`
```
/** struct ip_options - IP Options
 *
 * @faddr - Saved first hop address
 * @nexthop - Saved nexthop address in LSRR and SSRR (Loose Source Routing, Strict Source Routing)
 * @is_strictroute - Strict source route
 * @srr_is_hit - Packet destination addr was our one
 * @is_changed - IP checksum more not valid
 * @rr_needaddr - Need to record addr of outgoing dev
 * @ts_needtime - Need to record timestamp
 * @ts_needaddr - Need to record addr of outgoing dev
 */
struct ip_options {
	__be32		faddr;
	__be32		nexthop;
	unsigned char	optlen;
	unsigned char	srr;   //
	unsigned char	rr;    //Record Route
	unsigned char	ts;    // timestamp
	unsigned char	is_strictroute:1,
			srr_is_hit:1,
			is_changed:1,
			rr_needaddr:1,
			ts_needtime:1,
			ts_needaddr:1;
	unsigned char	router_alert;
	unsigned char	cipso;
	unsigned char	__pad2;
	unsigned char	__data[0];
};
```

##### Key functions
* `net/ipv4/ip_options.c`
```
ip_options_compile    // ingress

ip_options_build      // egress

__ip_options_echo

ip_options_fragment

```

#### ICMP

##### Initialization
```
static struct pernet_operations __net_initdata icmp_sk_ops = {
       .init = icmp_sk_init,
       .exit = icmp_sk_exit,
};
__init icmp_init
  register_pernet_subsys(&icmp_sk_ops)
```

##### Key functions
```
icmp_send
```

```
icmp_rcv

```

```
icmp_reply
```

```
icmp_redirect
```

```
icmp_unreach
```

```
icmp_echo
```

```
icmp_timestamp
```


#### ARP
* `neigh_resolve_output`

### Misc
`/proc/sys/net/ipv4/ip_forward` to enable/disable IP forwarding
`/proc/net/snmp`  ICMP statistics
