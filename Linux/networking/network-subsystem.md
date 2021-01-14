# Computer Network

## Bridge
[Anatomy of a Linux bridge](https://wiki.aalto.fi/download/attachments/70789083/linux_bridging_final.pdf)

[https://goyalankit.com/blog/linux-bridge](https://goyalankit.com/blog/linux-bridge)


## IPSec
[XFRM--A Kernel Implementation Framework of IPsec Protocol](https://programmer.ink/think/xfrm-a-kernel-implementation-framework-of-ipsec-protocol.html)

## Segmentation Offloads
[Segmentation Offloads @ Linux kernel doc](https://www.kernel.org/doc/html/latest/networking/segmentation-offloads.html)

### GSO
[GSO: Generic Segmentation Offload](https://wiki.linuxfoundation.org/networking/gso)
Many people have observed that a lot of the savings in TSO come from traversing the networking stack once rather than many times for each super-packet.
The key to minimising the cost in implementing this is to postpone the segmentation as late as possible. In the ideal world, the segmentation would occur inside each NIC driver.
Unfortunately this requires modifying each and every NIC driver so it would take quite some time. A much easier solution is to perform the segmentation just before the entry into the driver's xmit routine. This concept is called GSO: Generic Segmentation Offload.


## Lightweight
[Lightweight & flow based tunneling](https://lwn.net/Articles/650778/)


[BPF for lightweight tunnel encapsulation](https://lwn.net/Articles/705609/)
