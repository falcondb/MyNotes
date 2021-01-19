# Computer Network

## Bridge
[Anatomy of a Linux bridge](https://wiki.aalto.fi/download/attachments/70789083/linux_bridging_final.pdf)
_switch_
A software based switch, called a _bridge_. Network devices, called _switches_ are responsible for connecting several network links to each other, creating a local area network.

Conceptually, the major components of a network switch are a set of network ports, a control plane, a forwarding plane, and a MAC learning database. The control plane of a switch is typically used to run the Spanning Tree Protocol (STP) [15], that calculates a minimum spanning tree for the local area network, preventing physical loops from crashing the network.

For each unicast destination MAC address, the switch looks up the output port in the MAC database. If an entry is found, the frame is forwarded through the port further into the network. If an entry is not found, the frame is instead flooded from all other network ports in the switch, except the port where the frame was received

The Linux bridging module has two separate configuration interfaces exposed to the user-space of the operating system.
The first, ioctl interface to create and destroy bridges
The second, `sysfs` to tune parameters

_Forwarding database_
The forwarding database is an array of 256 elements, where each element is a singly linked list holding the forwarding table entries for the hash value. Unused entries in the forwarding table are cleaned up periodically by the br_cleanup function, that is invoked by
the garbage collection timer.

See bridge.md for code study


## IPSec
[XFRM--A Kernel Implementation Framework of IPsec Protocol](https://programmer.ink/think/xfrm-a-kernel-implementation-framework-of-ipsec-protocol.html)


## Lightweight tunneling
[Lightweight & flow based tunneling](https://lwn.net/Articles/650778/)


[BPF for lightweight tunnel encapsulation](https://lwn.net/Articles/705609/)
