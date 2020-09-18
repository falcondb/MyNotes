# Computer Network

## Bridge
[Anatomy of a Linux bridge](https://wiki.aalto.fi/download/attachments/70789083/linux_bridging_final.pdf)

[https://goyalankit.com/blog/linux-bridge](https://goyalankit.com/blog/linux-bridge)

## XDP
[Oracle post: The Power of XDP by Alan Maguire](https://blogs.oracle.com/linux/the-power-of-xdp)
>XDP comes in two flavours:
>>native XDP requires driver support, and packets are processed before sk_buffs are allocated. This allows us to realize the benefits of a minimal metadata descriptor. The hook comprises a call to bpf_prog_run_xdp, and after calling this function the driver must handle the possible return values - see below for a description of these. As an example, the bnxt_rx_pkt function calls bnxt_rx_xdp, which in turn verifies if an XDP program has been loaded for the RX ring, and if so sets up metadata buffer and calls bpf_prog_run_xdp. bnxt_rx_pkt is called directly from device polling functions and so is called via the net_rx_action for both interrupt processing and polling; in short we are getting our hands on the packet as soon as possible in the receive codepath.

>>generic XDP, where the XDP hooks are called from within the networking stack after the sk_buff has been allocated. Generic XDP allows us to use the benefits of XDP - though at a slightly higer performance cost - without underlying driver support. In this case bpf_prog_run_xdp is called as via netdev's netif_receive_generic_xdp function; i.e. after the skb has been allocated and set up. To ensure that XDP processing works, the skb has to be linearized (made contiguous rather than chunked in data fragments) - again this can cost performance.
