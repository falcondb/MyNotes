## BPF helper functions

### uapi/linux/bpf.h

* bpf_skb_get_tunnel_key
The `struct bpf_tunnel_key` is an object that generalizes theprincipal parameters used by various tunneling protocols into a single struct. This way, it can be used to easily make a decision based on the contents of the encapsulation header, "summarized" in this struct. In particular, it holds the IP address of the remote end (IPv4 or IPv6, depending on the case) in _key_`->remote_ipv4` or _key_ `->remote_ipv6`. Also, this struct exposes the _key_ `->tunnel_id`, which is generally mapped to a VNI (Virtual Network Identifier), making it programmable together with the `bpf_skb_set_tunnel_key` helper.
This can be used together with various tunnels such as VXLan, Geneve, GRE or IP in IP (IPIP)

Refer `samples/bpf/tc_l2_redirect*` for an usage
