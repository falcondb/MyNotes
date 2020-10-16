### Network interfaces
[Introduction to Linux interfaces for virtual networking](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking/)
* Bridge: It forwards packets between interfaces that are connected to it.
* Bonded interface: The Linux bonding driver provides a method for aggregating multiple network interfaces into a single logical “bonded” interface. The behavior of the bonded interface depends on the mode; generally speaking, modes provide either hot standby or load balancing services. [kernel doc](https://www.kernel.org/doc/Documentation/networking/bonding.txt)
* Team device:  provide a mechanism to group multiple NICs (ports) into one logical one (teamdev) at the L2 layer. a lockless (RCU) TX/RX path and modular design. [Diff between Team and Bonded](https://github.com/jpirko/libteam/wiki/Bonding-vs.-Team-features)
```
man 8 teamd
```
* VLAN: separates broadcast domains by adding tags to network packets. VLANs allow network administrators to group hosts under the same switch or between different switches. Totally, 12 bits for VID.
* VXLAN:
    * a tunneling protocol!!!
    * With a 24-bit segment ID, aka VXLAN Network Identifier (VNI), VXLAN allows up to 2^24 (16,777,216) virtual LANs
    * VXLAN encapsulates Layer 2 frames with a VXLAN header into a UDP-IP packet.
    * [XVLAN introduction](https://vincent.bernat.ch/en/blog/2017-vxlan-linux)
        * VLAN flexibility in multitenant segments, tenant workload can be placed across physical pods
        * Higher scalability, 24-bit segment ID
        * Improved network utilization, take complete advantage of Layer 3 routing
        * VXLAN defines a MAC-in-UDP encapsulation scheme where the original Layer 2 frame has a VXLAN header added and is then placed in a UDP-IP packet
        * directly get dhcp address from a common server

    * [XVLAN introduction](https://www.ciscopress.com/articles/article.asp?p=2999385&seqNum=3)

* MACVLAN:
    * with MACVLAN, you can bind a physical interface that is associated with a MACVLAN directly to namespaces, without the need for a bridge
    * Different modes in collecting the flow
    * each sub-interface will get unique mac and ip address and will be exposed directly in underlay network, in contrast, vlan devices share the same MAC address with the parent device
    * Mavlan sub-interfaces are not able to directly communicate with the parent interface, add another macvlan sub-interface and assign it to the host.
* IPVLAN:
    * IPVLAN is similar to MACVLAN with the difference being that the endpoints have the same MAC address.
    * IPVLAN L2 mode acts like a MACVLAN in bridge mode. In ipvlan l2 mode, each endpoint gets the same mac address but different ip address.
    * In IPVLAN L3 mode, the parent interface acts like a router.
* MACVTAP/IPVTAP: MACVTAP/IPVTAP replaces the combination of TUN/TAP and bridge drivers with a single module
* MACsec (802.1AE):
    * Security Tag, not found an authorized doc about its format.
    * Message authentication code (ICV) at the end of the encrypted data.
* VETH:
    * Packets transmitted on one device in the pair are immediately received on the other device. When either device is down, the link state of the pair is down.
* VCAN, VXCAN (Controller Area Network)
* IPOIB:
    * IP-over-InfiniBand protocol.
    * This transports IP packets over InfiniBand (IB)
    * datagram mode and connected mode
* NLMON: Netlink monitor device
```text
ip link add nlmon0 type nlmon
ip link set nlmon0 up
tcpdump -i nlmon0 -w nlmsg.pcap
```
* Dummy interface: virtual loopback interface, route packets through without actually transmitting them.
* IFB:

[An introduction to Linux virtual interfaces: Tunnels](https://developers.redhat.com/blog/2019/05/17/an-introduction-to-linux-virtual-interfaces-tunnels/)
* IPIP Tunnel: IP over IP, IPV4 only
    * IPV4 type == 4
    * eth head -> Outer IPV4 header (type == 4) -> Inner IPV4 header
    * IPIP tunnel supports both IP over IP and MPLS over IP.
    * local IP; remote IP
* SIT Tunnel: Simple Internet Transition
    * IPV4 type == 41 /* IPv6-in-IPv4 tunnelling */ ?! Not sure about it. TODO: Verify it later.
    * eth head -> Outer IPV4 header (type == 41?) -> Inner IPV6 header.
    * The main purpose is to interconnect isolated IPv6 networks, located in global IPv4 internet.
    * Mode any is used to accept both IP and IPv6 traffic

* ip6tnl Tunnel:
    * ip6tnl is an IPv4/IPv6 over IPv6 tunnel interface   
    * eth head -> Outer IPV6 header (type == ??) -> Inner IPV4/6 header.

* VTI: Virtual Tunnel Interface
    * This particular tunneling driver implements IP encapsulations, which can be used with xfrm to give the notion of a secure tunnel and then use kernel routing on top.


### Network header checksum
#### Computation of the Internet Checksum via Incremental Update
[RFC 1624](https://tools.ietf.org/html/rfc1624#page-1)
```
HC  - old checksum in header
C   - one's complement sum of old header
HC' - new checksum in header
C'  - one's complement sum of new header
m   - old value of a 16-bit field
m'  - new value of a 16-bit field

HC' = ~(C + (-m) + m')

if the one's complement sum overflows, add 1 back
```

### Type Of Service
* the second byte of the IPv4 header
* Traffic Class field in IPv6 packets, is an 8-bit differentiated services field (DS field) which consists of a 6-bit Differentiated Services Code Point (DSCP) field[3] and a 2-bit Explicit Congestion Notification (ECN) field. While Differentiated Services is somewhat backwards compatible with ToS, ECN is not.
* RFC 1349 introduced an additional "lowcost" field. lowdelay	throughput	reliability	lowcost, or [Linux TC PIOR Man page](https://man7.org/linux/man-pages/man8/tc-prio.8.html) Minimize delay (md), Maximize throughput (mt), Maximize reliability (mr),  Minimize monetary cost (mmc), Normal Service
* See [Linux TC PIOR Man page](https://man7.org/linux/man-pages/man8/tc-prio.8.html) for how Linux assigns package using TOS (md, mt, mr, mmc) to the pdisc bands.
* `tc qdisc show` shows the _priomap_ magic numbers as the band IDs for the _TOS_ values.

## LWN posts
[Van Jacobson's network channels](https://lwn.net/Articles/169961/) & [slides](http://www.lemis.com/grog/Documentation/vj/lca06vj.pdf)
