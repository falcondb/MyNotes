## BPF in Network stack

#### Device
* `__bpf_redirect`
```
__bpf_redirect
  with Mac? __bpf_redirect_common : __bpf_redirect_no_mac
```

```
__bpf_redirect_no_mac
  skb_pop_mac_header
  skb_reset_mac_len
  BPF_F_INGRESS ?
	       __bpf_rx_skb_no_mac(dev, skb) : __bpf_tx_skb(dev, skb)
```

```
__bpf_redirect_common
  BPF_F_INGRESS ?
      __bpf_rx_skb(dev, skb) : __bpf_tx_skb(dev, skb)
```

```
__bpf_rx_skb  ==> dev_forward_skb

```

```
__bpf_tx_skb
```

```
__bpf_rx_skb_no_mac
  ____dev_forward_skb
  netif_rx

```

```
__bpf_tx_skb_no_mac
```
