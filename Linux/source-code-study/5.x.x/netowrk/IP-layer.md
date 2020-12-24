## IP layer

### ip_input.c
* `ip_rcv`

### ip_output.c
* `ip_output`


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
