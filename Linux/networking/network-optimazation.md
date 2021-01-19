[Scaling in the Linux Networking Stack](https://github.com/torvalds/linux/blob/v3.13/Documentation/networking/scaling.txt#L364-L422)

_RSS: Receive Side Scaling_
Contemporary NICs support multiple receive and transmit descriptor queues. On reception, a NIC can send different packets to different queues to distribute processing among CPUs.
The NIC distributes packets by applying a filter to each packet that assigns it to one of a small number of logical flows. Packets for each flow are steered to a separate receive queue, which in turn can be processed by separate CPUs. This mechanism is
generally known as _Receive-side Scaling_.

Each receive queue has a separate IRQ associated with it. The NIC triggers
this to notify a CPU when new packets arrive on the given queue. The active mapping
of queues to IRQs can be determined from `/proc/interrupts`.
To manually adjust the IRQ affinity of each interrupt see `Documentation/IRQ-affinity.txt`. Some systems will be running `irqbalance`, a daemon that dynamically optimizes IRQ assignments and as a result may override any manual settings.

RSS should be enabled when latency is a concern or whenever receive
interrupt processing forms a bottleneck. Spreading load between CPUs
decreases queue length.

_RPS: Receive Packet Steering_
Receive Packet Steering (RPS) is logically a software implementation of RSS.
RSS selects the queue and hence CPU that will run the hardware interrupt handler, RPS selects the CPU to perform protocol processing above the interrupt handler.
RPS has some advantages over RSS:
  1) it can be used with any NIC,
  2) software filters can easily be added to hash over new protocols,
  3) it does not increase hardware device interrupt rate (although it does introduce inter-processor interrupts (IPIs)).

The hash is saved in `kb->hash` and can be used elsewhere in the stack as a hash of the packet’s flow. At the end of the bottom half routine, IPIs are sent to any CPUs for which packets have been queued to their backlog queue. The IPI wakes backlog processing on the remote CPU, and any queued packets are then processed up the networking stack.

_RFS: Receive Flow Steering_
The goal of RFS is to increase datacache hitrate by steering kernel processing of packets to the CPU where the application thread consuming the packet is running.

In RFS, packets are not forwarded directly by the value of their hash, but the hash is used as index into a flow lookup table. This table maps flows to the CPUs where those flows are being processed. The CPU recorded in each entry is the one which last processed the flow. If an entry does not hold a valid CPU, then packets mapped to that entry are steered using plain RPS.

`rps_sock_flow_table` is a global flow table that contains the desired CPU for flows: the CPU that is currently processing the flow in userspace. Each table value is a CPU index that is updated during calls to `recvmsg` and `sendmsg`

When the scheduler moves a thread to a new CPU while it has outstanding receive packets on the old CPU, packets may arrive out of order. To avoid this, RFS uses a second flow table to track outstanding packets for each flow: rps_dev_flow_table is a table specific to each hardware receive queue of each device. See the design details about avoid out-of-order packets in the Documentation.

_Accelerated RFS_
A hardware-accelerated load balancing mechanism that uses soft state to steer flows based on where the application thread consuming the packets of each flow is running.


_Transmit Packet Steering XPS_
Transmit Packet Steering is a mechanism for intelligently selecting which transmit queue to use when transmitting a packet on a multi-queue device. This can be accomplished by recording two kinds of maps, either a mapping of CPU to hardware queue(s) or a mapping of receive queue(s) to hardware transmit queue(s).

1. XPS using CPUs map
  The goal of this mapping is usually to assign queues exclusively to a subset of CPUs. This choice provides two benefits. First, contention on the device queue lock is significantly reduced since fewer CPUs contend for the same queue. Secondly, cache miss rate on transmit completion is reduced, in particular for data cache lines that hold the `sk_buff` structures.
2. XPS using receive queues map
  This mapping is used to pick transmit queue based on the receive queue(s) map configuration set by the administrator. A set of receive queues can be mapped to a set of transmit queues. This is useful for busy polling multi-threaded workloads where there are challenges in associating a given CPU to a given application thread. It has benefits in keeping the CPU overhead low. Transmit completion work is locked into the same queue-association that a given application is polling on. This avoids the overhead of triggering an interrupt on another CPU.


[Segmentation Offloads](https://www.kernel.org/doc/html/latest/networking/segmentation-offloads.html)
_TCP Segmentation Offload_
TCP segmentation allows a device to segment a single frame into multiple frames with a data payload size specified in `skb_shinfo()->gso_size`.
TCP segmentation is dependent on support for the use of partial checksum offload. For this reason TSO is normally disabled if the Tx checksum offload for a given device is disabled.
For IPv4 segmentation we support one of two types in terms of the IP ID. The default behavior is to increment the IP ID with every segment. If the GSO type SKB_GSO_TCP_FIXEDID is specified then we will not increment the IP ID and all segments will use the same IP ID.



_IPIP, SIT, GRE, UDP Tunnel, and Remote Checksum Offloads_



_GSO: Generic Segmentation Offload_
[GSO: Generic Segmentation Offload](https://wiki.linuxfoundation.org/networking/gso)
Many people have observed that a lot of the savings in TSO come from traversing the networking stack once rather than many times for each super-packet.
The key to minimising the cost in implementing this is to postpone the segmentation as late as possible. In the ideal world, the segmentation would occur inside each NIC driver.
Unfortunately this requires modifying each and every NIC driver so it would take quite some time. A much easier solution is to perform the segmentation just before the entry into the driver's xmit routine. This concept is called GSO: Generic Segmentation Offload.

_Generic Receive Offload GRO_
Generic receive offload is the complement to GSO. Ideally any frame assembled by GRO should be segmented to create an identical sequence of frames using GSO, and any sequence of frames segmented by GSO should be able to be reassembled back to the original by GRO.

_Partial Generic Segmentation Offload_
Partial generic segmentation offload is a hybrid between TSO and GSO. What it effectively does is take advantage of certain traits of TCP and tunnels so that instead of having to rewrite the packet headers for each segment only the inner-most transport header and possibly the outer-most network header need to be updated.

_SCTP acceleration with GSO_

TOBESTUDIED
