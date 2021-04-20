### network information from sysfs and proc filesystems
* link level information in `/sys/class/net`, e.g., `/sys/class/net/$DEV/address` MAC address; `/sys/class/net/$DEV/dev_id` if_index; mtu; type; queues; statistics;
* IP level information in `/proc/net` or `/proc/$PID/net`
  * `/proc/net/nf_conntrack` conntract connections; `/proc/net/dev` statistics; `/proc/net/fib_trie` IP addresses; TCP/UDP/ICMP statistics; `/proc/$PID/fd`

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
    `modprobe ipip`
    [Tunneling: IPIP Encapsulation](https://www.oreilly.com/library/view/wireless-hacks/0596005598/ch04s13.html)
    > an IP tunnel is much like a VPN, except that not every IP tunnel involves encryption. A machine that is “tunneled” into another network has a virtual interface configured with an IP address that isn’t local, but exists on a remote network

    > If you want to perform simple IP-within-IP tunneling between two machines, you might try IPIP. It is also only capable of tunneling unicast packets; if you need to tunnel multicast traffic, take a look at GRE tunneling


* SIT Tunnel: Simple Internet Transition
    * IPV4 type == 41 // IPv6-in-IPv4 tunnelling?! Not sure about it. TODO: Verify it later.
    * eth head -> Outer IPV4 header (type == 41?) -> Inner IPV6 header.
    * The main purpose is to interconnect isolated IPv6 networks, located in global IPv4 internet.
    * Mode any is used to accept both IP and IPv6 traffic

* ip6tnl Tunnel:
    * ip6tnl is an IPv4/IPv6 over IPv6 tunnel interface   
    * eth head -> Outer IPV6 header (type == ??) -> Inner IPV4/6 header.

* VTI: Virtual Tunnel Interface
    * This particular tunneling driver implements IP encapsulations, which can be used with xfrm to give the notion of a secure tunnel and then use kernel routing on top.

[The Linux Networking Architecture: Design and Implementation of Network Protocols in the Linux Kernel](https://freecomputerbooks.com/The-Linux-Networking-Architecture.html)
* Synchronization
  * software interrupt
    a software interrupt is scheduled for execution by an activity of the kernel and has to wait until it is called by the scheduler. Software interrupts scheduled for execution are started by the function `do_softirq`. A soft IRQ is activated by `__cpu_raise_softirq`. This occurs currently only when a system call in `schedule()` or a hardware interrupt in `do_IRQ()` terminates.

    A software interrupt can run concurrently in several processors, implemented reentrantly.
    A software interrupt cannot interrupt itself while running on a processor.
    A software interrupt can be interrupted during its handling on a processor only by a hardware interrupt.

  * Tasklets
    The function `tasklet_schedule(&tasklet_struct)` can be used to schedule a tasklet for execution. A tasklet is run only once, even if it was scheduled for execution several times. `tasklet_disable()` can be used to stop a tasklet from running, even when it is scheduled for execution
    A tasklet can run on one processor only at any given time.
    Different tasklets can run on several processors concurrently.

  * Bottom Halfs
    Only one bottom half can run concurrently on all processors of a system at one time.

  * Bit operations
    Atomic bit operations form the basis for the locking concepts spinlocks and semaphores. `atomic_t` for integer. All of the atomic operations are implemented by _one single machine command_.

  * Spinlocks
    See locking.md

  * Read-write Spinlocks
    RW spinlock functions come in different variants with regard to how they handle _interrupts and bottom halves_

* Kernel module
  * `init_module` & `cleanup_module`
  * `insmod`
    * `sys_create_module` allocates memory space to accommodate the module in the kernel address space
    * `sys_get_kernel_syms` returns the kernel's symbol table to resolve the missing references within the module to kernel symbols
    * `sys_init_module` copies the module's object code into the kernel address space and calls the module's initialization function
  * Loading Modules Automatically  
    Normally, the kernel generates an error message when a resource or a specific driver is not registered. You can ask for this component in advance by use of the kernel function `request_module()`. To use this function, you have to first activate the option Kernel Module Loader when configuring the kernel. `request_module()` will then try to use the `modprobe` command to automatically reload the desired module . You can select such options in the file `/etc/modules.conf`.
* Timing
  In the Pentium processor, a 64-bit-wide _TSC_ (Time Stamp Counter) register. Nevertheless, there is a certain inaccuracy when measuring with the _TSC_ register, because it takes a few clocks (approx. ten) to read the register.


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
* Traffic Class field in IPv6 packets, is an 8-bit differentiated services field (DS field) which consists of a 6-bit _Differentiated Services Code Point (DSCP)_ field and a 2-bit _Explicit Congestion Notification (ECN)_ field. While Differentiated Services is somewhat backwards compatible with ToS, ECN is not.
* RFC 1349 introduced an additional "lowcost" field. lowdelay	throughput reliability	lowcost, or [Linux TC PIOR Man page](https://man7.org/linux/man-pages/man8/tc-prio.8.html) Minimize delay (md), Maximize throughput (mt), Maximize reliability (mr),  Minimize monetary cost (mmc), Normal Service
* See [Linux TC PIOR Man page](https://man7.org/linux/man-pages/man8/tc-prio.8.html) for how Linux assigns package using TOS (md, mt, mr, mmc) to the pdisc bands.
* `tc qdisc show` shows the _priomap_ magic numbers as the band IDs for the _TOS_ values.

## LWN posts
[Van Jacobson's network channels](https://lwn.net/Articles/169961/) & [slides](http://www.lemis.com/grog/Documentation/vj/lca06vj.pdf)


## TPC
[RFC 793](https://tools.ietf.org/html/rfc793)
