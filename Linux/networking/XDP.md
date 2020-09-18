##XDP-tutorial
[XDP-tutorial](https://github.com/xdp-project/xdp-tutorial)
[Video talk: XDP closer integration with network stack @ Kernel Receipts 2019](https://www.youtube.com/watch?v=JgJQpcaaCR8)

##XDP Papers
[The eXpress Data Path: Fast Programmable Packet Processing in
the Operating System Kernel](http://borkmann.ch/paper/2018_xdp.pdf)

## miscellaneous
* To attach egress path, eBPF has to be loaded using `tc` or `bpf syscall` and attached in the egress path.
`tc` can load and attach eBPF programs to a qdisc just like any other action. 
* Differences between the XDP and tc API, the default section name differs, the argument (struct __sk_buff vs struct xdp_md), and the returned values (TC_ACT_XXX vs XDP_XXX).
```text
## Load BPF to egress path by tc
# tc qdisc add dev eth0 clsact
# tc filter add dev eth0 egress filterchall action bpf object-file bpf.o
```
* TODO: the diff between `bpf_htons` defined in `bpf_endian.h` and the `__constant_htons` defined in `linux/byteorder/little_endian.h`


