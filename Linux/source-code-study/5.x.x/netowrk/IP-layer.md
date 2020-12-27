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

* `dev_queue_xmit`


#### ARP
* `neigh_resolve_output`

### Misc
`/proc/sys/net/ipv4/ip_forward` to enable/disable IP forwarding
