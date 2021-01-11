## NAPI

[NAPI @ Linux linuxfoundation](https://wiki.linuxfoundation.org/networking/napi) & [Driver porting: Network drivers @ LWN.net](https://lwn.net/Articles/30107/)
NAPI (“New API”) is an extension to the device driver packet processing framework, which is designed to improve the performance of high-speed networking
_NAPI works through:_
* Interrupt mitigation: High-speed networking can create thousands of interrupts per second, all of which tell the system something it already knew: it has lots of packets to process. NAPI allows drivers to run with (some) interrupts disabled during times of high traffic, with a corresponding decrease in system load.

* Packet throttling: When the system is overwhelmed and must drop packets, it's better if those packets are disposed of before much effort goes into processing them. NAPI-compliant drivers can often cause packets to be dropped in the network adaptor itself, before the kernel sees them at all.

* More careful packet treatment, with special care taken to avoid reordering packets.

New drivers should use NAPI if the hardware can support it.
Make some changes to your driver's interrupt handler. If your driver has been interrupted because a new packet is available, that packet should not be processed at that time. Instead, your driver should disable any further “packet available” interrupts and tell the networking subsystem to poll your driver shortly to pick up all available packets.
Create a poll() method for your driver; it's job is to obtain packets from the network interface and feed them into the kernel. The poll() function should process all available incoming packets, much as your interrupt handler might have done in the pre-NAPI days
- Packets should not be passed to netif_rx(); instead, use: `netif_receive_skb(struct sk_buff *skb)` in `net/core/dev.c`

_Hardware Architecture_
NAPI, however, requires the following features to be available:
- DMA ring or enough RAM to store packets in software devices.
- Ability to turn off interrupts or maybe events that send packets up the stack.

[NAPI polling in kernel threads @ LWN.net](https://lwn.net/Articles/833840/)
