### network interfaces
[Introduction to Linux interfaces for virtual networking](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking/)
* Bridge: It forwards packets between interfaces that are connected to it.
* Bonded interface: The Linux bonding driver provides a method for aggregating multiple network interfaces into a single logical “bonded” interface. The behavior of the bonded interface depends on the mode; generally speaking, modes provide either hot standby or load balancing services. [kernel doc](https://www.kernel.org/doc/Documentation/networking/bonding.txt)
* Team device:  provide a mechanism to group multiple NICs (ports) into one logical one (teamdev) at the L2 layer. a lockless (RCU) TX/RX path and modular design. [Diff between Team and Bonded](https://github.com/jpirko/libteam/wiki/Bonding-vs.-Team-features)
```
man 8 teamd
```
* VLAN: separates broadcast domains by adding tags to network packets. VLANs allow network administrators to group hosts under the same switch or between different switches. Totally, 12 bits for VID.
* VXLAN: a tunneling protocol!!! With a 24-bit segment ID, aka VXLAN Network Identifier (VNI), VXLAN allows up to 2^24 (16,777,216) virtual LANs
*
*


[An introduction to Linux virtual interfaces: Tunnels](https://developers.redhat.com/blog/2019/05/17/an-introduction-to-linux-virtual-interfaces-tunnels/)
