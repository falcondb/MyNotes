## Netlink

[Netlink and usespace, netlink example](https://blog.csdn.net/qq_18144747/article/details/98179350)

### Key functions

* `af_netlink.c`

```
netlink_rcv_skb
  cb    // the passed in callback

  netlink_ack if NLM_F_ACK

```
