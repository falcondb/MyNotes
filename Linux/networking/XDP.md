##XDP-tutorial
[XDP-tutorial](https://github.com/xdp-project/xdp-tutorial)
[Video talk: XDP closer integration with network stack @ Kernel Receipts 2019](https://www.youtube.com/watch?v=JgJQpcaaCR8)

##XDP Papers
[The eXpress Data Path: Fast Programmable Packet Processing in
the Operating System Kernel](http://borkmann.ch/paper/2018_xdp.pdf)

##AF_XDP or XSK
[kernel.org AF_XDP overview](https://www.kernel.org/doc/html/v4.18/networking/af_xdp.html#)
> UMEM is a region of virtual contiguous memory, divided into equal-sized frames. An UMEM is associated to a netdev and a specific queue id of that netdev.
> created by XDP_UMEM_REG setsockopt. bound to a netdev and queue id, via the bind().

> An AF_XDP is socket linked to a single UMEM, but one UMEM can have multiple AF_XDP sockets. XDP_SHARED_UMEM flag in struct sockaddr_xdp, passing the file descriptor of A to struct sockaddr_xdp member sxdp_shared_umem_fd.

> The UMEM has two single-producer/single-consumer rings, that are used to transfer ownership of UMEM frames between the kernel and the user-space application.

> The UMEM uses two rings: Fill and Completion. Each socket associated with the UMEM must have an RX queue, TX queue or both. Say, that there is a setup with four sockets. Then there will be one Fill ring, one Completion ring, four TX rings and four RX rings.

> The Fill ring is used to transfer ownership of UMEM frames from user-space to kernel-space.
> The Completion Ring is used transfer ownership of UMEM frames from kernel-space to user-space.

> The RX ring is the receiving side of a socket. Each entry in the ring is a struct xdp_desc descriptor. The descriptor contains UMEM offset and the length of the data.

> The TX ring is used to send frames. To start the transfer a sendmsg() system call is required. The user application produces struct xdp_desc descriptors to this ring.



[Accelerating networking with AF_XDP](https://lwn.net/Articles/750845/)
The post here well explains how _AF_XDP_ should be configured in userland (socket, bind, UMEM, fill Q, comp Q, RX Q, TX Q, BPF_MAP_TYPE_XSKMAP), the functionalities of 4 types of queue, the registration of the queue fds with UMEM and socket, and the package routing in kernel XDP to the queues.
> AF_XDP is intended to connect the XDP path through to user space.
[BPF and XDP Reference Guide at cilium.io](https://docs.cilium.io/en/latest/bpf/)

UMEM is a user space address area which holds the network packages. The packages to TX are placed in TX queue as descriptors, after placing the packages from TX queue to UMEM, the ownership is changed to kernel, when the packages are sent out, a descriptor will be placed in the completion queue. The packages to RX are registered as a descriptor and placed in the fill queue, when the packages are received in the kernel, the XDP program forwards the packages to one of the receive queues (if multiple receive queues are registered with the UMEM) based on the BPF_MAP_TYPE_XSKMAP using the fd of the socket with the receive queue.


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
