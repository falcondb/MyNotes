## Network package processing
### ingress
Key steps in ingress path
  - NIC receives a frame from wire, _DMA_ to memory, raise a hardware interrupt, hardware interrupt handler in device driver adds the frame to device's `input_pkt_queue`, and the device to `poll_list` of the percpu `softnet_data`. The `poll_list` includes the devices with pending received frames. The driver raises SoftIRQ with handler `net_rx_action`. `net_rx_action` goes through the devices in `poll_list` and calls `napi_poll` on the device. `napi_poll` then calls the poll function registered with the device. The poll function is on the driver side and it eventually calls `netif_receive_skb`, which starts to process the frame. `netif_receive_skb` may enqueue the frame to a different CPU's `softnet_data` through _RPS(Receive Packet Steering see below)_, otherwise, calls `__netif_receive_skb ==> __netif_receive_skb_one_core ==> __netif_receive_skb_core`. `__netif_receive_skb_core` handles _frame taps_, _CONFIG_NET_INGRESS_, for _VLAN_, gets the device from _VLAN ID_ and pull out the _VLAN_ header from _SKB_, then calls `deliver_skb`. `deliver_skb` calls the network protocol's handlers, e.g., `ip_rcv` for IPv4 package.

[Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/)

_Register an interrupt handler_
different methods a device can use to signal an interrupt: MSI-X, MSI(Message Signalled Interrupts), and legacy interrupts. MSI-X interrupts are the preferred method, especially for NICs that support multiple RX queues. This is because each RX queue can have its own hardware interrupt assigned, which can then be handled by a specific CPU (`/proc/irq/IRQ_NUMBER/smp_affinity`). If MSI-X is unavailable, MSI still presents advantages over legacy interrupts and will be used by the driver if the device supports it.

_SoftIRQs_
The softirq system in the Linux kernel is a mechanism for executing code outside of the context of an interrupt handler implemented in a driver.
The softirq system can be imagined as a series of kernel threads that run handler functions which have been registered for different softirq events.

_Generic Receive Offloading (GRO)_interrupt.md
Generic Receive Offloading (GRO) is a software implementation of a hardware optimization that is known as Large Receive Offloading (LRO). The main idea behind both methods is that reducing the number of packets passed up the network stack by combining “similar enough” packets together can reduce CPU usage. GRO was introduced as an implementation of LRO in software, but with more strict rules around which packets can be coalesced.

A list of GRO offload filters is traversed to allow the higher level protocol stacks to act on a piece of data which is being considered for GRO. This is done so that the protocol layers can let the network device layer know if this packet is part of a network flow that is currently being receive offloaded and handle anything protocol specific that should happen for GRO.

Upper layer protocols' callback functions determines if the packets should be merged.

_Receive Packet Steering (RPS)_
Some NICs support multiple queues at the hardware level. This means incoming packets can be DMA’d to a separate memory region for each queue, with a separate NAPI structure to manage polling this region, as well. Thus multiple CPUs will handle interrupts from the device and also process packets.
This feature is typically called Receive Side Scaling (RSS). Receive Packet Steering (RPS) is a software implementation of RSS. Since it is implemented in software, this means it can be enabled for any NIC, even NICs which have only a single RX queue.

RPS works by generating a hash for incoming data to determine which CPU should process the data. The data is then enqueued to the per-CPU receive network backlog to be processed. An Inter-processor Interrupt (IPI) is delivered to the CPU owning the backlog. This helps to kick-start backlog processing if it is not currently processing data on the backlog.

_Receive Flow Steering (RFS)_
Receive flow steering (RFS) is used in conjunction with RPS. RPS attempts to distribute incoming packet load amongst multiple CPUs, but does not take into account any data locality issues for maximizing CPU cache hit rates. You can use RFS to help increase cache hit rates by directing packets for the same flow to the same CPU for processing.


_IP Layer_
if you have numerous or very complex netfilter or iptables rules, those rules will be executed in the softirq context and can lead to latency in your network stack.



### egress
Key steps in egress path
  - Create a socket with address family (`AF_INET, AF_INET6`), socket type (`SOCK_STREAM, SOCK_DGRAM, SOCK_RAW`) and protocol (`IPPROTO_TCP, IPPROTO_UDP, IPPROTO_RAW`). In kernel, handlers for address families are registered at `struct proto_ops` and `struct proto` in `struct inet_protosw inetsw_array[]`.
  - Switch to kernel model from system call, take _UDP_ as an example of transport layer (_TCP_ and _SCTP_ works on a lot on fragmentation, thus, less work for network layer, e.g., _TCP_ calls `dst_output`, `ip_queue_xmit` directly). The system call for UDP is `sock_sendmsg`, which calls `security_socket_sendmsg` or `sock_sendmsg_nosec ==> sock->ops->sendmsg`. For `SOCK_DGRAM`, `inet_sendmsg` is registered as `sendmsg`. `sendmsg` just calls `sk->sk_prot->sendmsg`, which points to `udp_sendmsg` for `IPPROTO_UDP`.
  - In `udp_sendmsg`, accumulating more messages and calls `ip_append_data` for _UDP corking_ (refer to _Understand Linux Network Internals_). Next handling socket control message, IP options (SRR and TOS), Multicast, DST cache check, ARP cache confirm. Non corking, `ip_make_skb` and `udp_send_skb`; corking case, `ip_append_data` and `udp_flush_pending_frames ==> udp_flush_pending_frames; udp_send_skb` (refer to _Understand Linux Network Internals_).
  - `udp_send_skb` handles _GSO_, _UDPLITE_, adds checksum in transport header, then calls `ip_send_skb`.
  - TCP path:
  `tcp_push_one ==> tcp_write_xmit ==> __tcp_transmit_skb`. `__tcp_transmit_skb` builds up the TCP header with its payload, then calls the `icsk_af_ops queue_xmit`, which points to `ip_queue_xmit/__ip_queue_xmit` for IPv4. `__ip_queue_xmit` goes through the routing system and builds up the IP header, finally calls ip_local_out. 

  - `ip_send_skb ==> ip_local_out ==> __ip_local_out`,  `__ip_local_out` makes a _netfilter_ hook call, `nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT,..., dst_output)`.
  - `skb_dst(skb)->output` points to `ip_output` or `ip_mc_output`. `ip_output` makes another _netfilter_ hook call, `NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING,..., ip_finish_output)`. `ip_mc_output` makes a _netfilter_ hook call, `NF_HOOK(NFPROTO_IPV4, NF_INET_POST_ROUTING,ip_mc_finish_output)` or `NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING,..., ip_finish_output)`.
  - `ip_finish_output` runs `BPF_CGROUP_RUN_PROG_INET_EGRESS`, then `__ip_finish_output`, which first handles _XFRM_ policy, _GSO_ (`ip_finish_output_gso`), `ip_fragment` if necessary, or `ip_finish_output2`.
  - `ip_finish_output2` calls `ip_neigh_for_gw`, which looks up the neighbour cache or quries neighbour table, then `neigh_output`.
  - `neigh_output` calls `neigh_hh_output` using neighbour cache with the link layer header or the slow route `struct neighbour.output`
  - `struct neighbour.output` can be set to different handlers, `neigh_direct_output`, `neigh_resolve_output` according to destination state (_NUD_). Eventually, they will call `dev_queue_xmit`

  - OK, entering _TC_ and Link layer from `dev_queue_xmit`. For non soft-device, `netdev_core_pick_tx ==> __dev_xmit_skb`
[Monitoring and Tuning the Linux Networking Stack: Sending Data](https://blog.packagecloud.io/eng/2017/02/06/monitoring-tuning-linux-networking-stack-sending-data/)

_IP Layer_
If you have numerous or very complex netfilter or iptables rules, those rules will be executed in the CPU context of the user process which initiated the original sendmsg call. If you have CPU pinning set up to restrict execution of this process to a particular CPU (or set of CPUs), be aware that the CPU will spend system time processing outbound iptables rules.

_Transmit Packet Steering (XPS)_
Transmit Packet Steering (XPS) is a feature that allows the system administrator to determine which CPUs can process transmit operations for each available transmit queue supported by the device. The aim of this feature is mainly to avoid lock contention when processing transmit requests. Other benefits like reducing cache evictions and avoiding remote memory access on NUMA machines are also expected when using XPS.

_Dynamic Queue Limits (DQL)_
Network data spends a lot of time sitting queues at various stages as it moves closer and closer to the device for transmission. As queue sizes increase, packets spend longer sitting in queues not being transmit.
The dynamic queue limit (DQL) system is a mechanism that device drivers can use to apply back pressure to the networking system to prevent too much data from being queued for transmit when the device is unable to transmit.
To use this system, network device drivers need to make a few simple API calls during their transmit and completion routines. The DQL system internally will use an algorithm to determine when sufficient data is in transmit. Once this limit is reached, the transmit queue will be temporarily disabled. This queue disabling is what produces the back pressure against the networking system. The queue will be automatically re-enabled when the DQL system determines enough data has finished transmission.

[slides about the DQL system](https://blog.linuxplumbersconf.org/2012/wp-content/uploads/2012/08/bql_slide.pdf)

_Transmit completions_
Devices may raise the same interrupt for receiving packets that it raises to signal that a packet transmit has completed. If it does, the `NET_RX` softirq runs to process both incoming packets and transmit completions.
