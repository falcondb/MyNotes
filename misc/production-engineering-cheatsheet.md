##trouble shooting methodology
###Investigation:
- Any possible relevant events, such as hardware, kernel, platform software upgrade, unexpected workload, network traffic congestion, operation errors.
- Environment: own data center, cloud, customer system, network (network topology (fat tree), virtual, routing, tunneling), CPU/Memory/Network model, the runtime, bare metal, VM, container, hardware. Application characteristics (CPU intensive, data intensive, IO/network intensive), Application architecture. Application binary with symbol table, optimized.
- Pattern: repeatable or one time, event period, one particular environment or almost everywhere.

###Trouble shouting strategy and Plan:
- The scope of the event: single machine or large scope
- The computer resources: CPU, memory, network, storage, or configuration.
- The software level: user space or application, kernel, hardware

###Executions:
####Examination:
  - System: uname, kernal configuration file (/boot/config), sysctl, lsmod, bootparam
  - Storage: mount, df, du, lsof(open /proc/$pid/fd, fdinfo), fuser (open /proc/$pid/fd, fdinfo), dumpe2fs, fdisk
  - CPU: uptime, ps, top, pidstat,  /proc /sys
  - Memory: sysctl, /proc/memoryinfo, free, numactl
    - `/sys/devices/system/node/` for status, `/sys/kernel/mm/` for the features.
  - Network: make sure enter the right namespace, ip a:l:route, bridge/brctl, ethtool, iptables (nftables), ebtables, tc, ss (netlink), `/sys/class/net`, nc/socat, dig
  - Cgroups: where, for what, for whom, current status
  - Namepces: `nsenter` `unshare` (`clone`),
####Monitoring
  - Utilization, Saturation, Error (dmesg, /var/log/, /sys/, journalctl/systemd),
  - sar, iostat(/proc/diskstats, /proc/stat), vmstat (/proc/vmstat), mpstat, /proc /sys, perf (syscall) ftrace eBPF,
  - Process: prtstat(proc/$PID/stat), pidstat, pstree

#### proc fs
#####/proc/stat
  - CPU: iowait, irq, softirq
  - intr: counts of interrupts
  - ctxt: number of context switches across all CPUs
  - btime: boot time
  - processes: number of processes and threads created
  - procs_running: number of processes currently running on CPUs
  - procs_blocked: number of processes currently blocked, waiting for I/O to complete

#####/proc/uptime
  - #1 up time in seconds, #2 idle time

#####/proc/softirq
  - Columns: softirq types, including NET_RX, NET_TX, BLOCK, TASKLET, SCHED, and RCU

#####/proc/interrupts
  - #1 IRQ number, #2 event count, #3 device that is located at that IRQ
  - e.g., TLB shootdowns, Function call interrupts(INT ??), Rescheduling interrupts(spread workload to other processors by waking up it and run sched)

#####/proc/$PID/stat
  [pid proc manual](https://man7.org/linux/man-pages/man5/proc.5.html)
  - #10-13 page faults
  - #14 utime, #15 stime, #16 waited-for children time in user mode, #17 waited-for children time in kernel mode
  - #20 num_threads
  - #22 The time the process started after system boot
  - #23 vsize Virtual memory size, #24 rss Resident Set Size
  - #26-27 address of code area, stack area, `startstack` start address of its stack, #45-46 address of data area, #47 `start_brk` heap can expand, `kstkesp` current stack pointer ESP, `kstkeip` current instruction pointer EIP
  - #31-34 signals status #38 exit signal to its parent
  - #48-49 `arg_start` `arg_end`, #50-51 `env_start` `env_end`, #52 `exit_code` thread exit code

#####/proc/net
  - `dev`: RX, TX statistics
  - `nf_conntrack` or `ip_conntrack`
  - `route` `ipv6_route`: route table, IP in little endian?!
  - `tcp` `tcp6` `udp` `udp6`: (Documentation/networking/proc_net_tcp.txt)[https://www.kernel.org/doc/Documentation/networking/proc_net_tcp.txt]
  - `sockstat` `sockstat6`: stats in socket level
  - `netlink`: sk, pid, inode
  - `netstat`: detailed stats
  - `softnet_stat`: Per CPU, #1 packet_process; #2 packet_drop; #3 time_squeeze; #8 cpu_collision; #9: received_rps; #10: flow_limit_count
  - `fib_trie` `fib_triestat`: forwarding tables and their stats
  - `dev_mcast`, `igmp`
  - `arp`: ARP table


#####/proc/sys
  - core
    - BPF settings, xfrm settings, flow limits
  - ipv4 and ipv6: their Configurations
  - netfilter: [Documentation/networking/nf_conntrack-sysctl.txt](https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt)
  - bridge: bridge configurations

#####/proc/filesystems
  available fs systems

#####/proc/partitions
  - major, minor, #blocks, name

#####/proc/diskstats
  Documentation/iostats.txt
  - # of reads completed, # of reads merged, # of sectors read, # of milliseconds spent reading
  - # of writes completed, # of writes merged, # of sectors written, # of milliseconds spent writing
  - # of I/Os currently in progress, # of milliseconds spent doing I/Os

#### sys fs
#####/sys/class/net
  - $DEVNAME
    - Configurations of the device, such as IP address, ifindex, mtu
    - /statistics: rx and tx stats, including rx_bytes, rx_XXX_errors

#####/sys/kernel
  - cgroup
  - debug: kprobes, tracing (ftrace)
  - mm: hugepages, transparent_hugepage
  - slab: configurations for each slab

#####/sys/devices/system/cpu/cpuX/
  Documentation/ABI/testing/sysfs-devices-system-cpu
  - topology: core_cpus, core_siblings, die_cpus_list
  - crash_notes: crash_notes: the physical address of the memory that holds the	note of cpu#

#####/sys/dev/block/XXX:YY
  detailed information of a storage device
  - stat, capability, mq, queue, power, trace

####/etc
#####/etc/sysconfig
  - configuration files of different services, such as init, sshd, iptables

####logs
  - /var/log
    - message: serive system logs
    - dmesg: kernel ring buffer
    - daemon.log: background daemon log
    - kern.log: kernel logs

  - /dev/kmsg: kernel's printk buffer



#### Tools using socket
  - Command goes through socket _AF_NETLINK_
    - ip, bridge, nft (nftables), tc
  - Command goes through socket _AF_INET_
    - ethtool, iptables, ifconfig, arp
  - Command goes through socket _AF_UNIX_
    - brctl

####Tracing
  stace, ltrace, ftrace, eBPF, mtrace

####Sampling
  perf, ftrace, eBPF, tcpdump

## Concepts

### System call
- `INT 0x80`: interrupt handler `ia32_syscall`. syscall number to EAX and other parameters to EBX, ECX, EDX. pushes the registers onto the kernel stack. Push `SS ESP EFLAGS CS EIP` to stack before entering kernel mode. When returns, `iret`
- `SYSCALL/SYSRET` 64-bit: `IA32_LSTAR` with the syscall handler. 6 registers for parameters. RAX for syscall number and return value. RCX for the resume IP after syscall.
- `SYSENTER/SYSEXIT` IA-32: `IA32_SYSENTER_CS /EIP /ESP`
- `__kernel_vsyscall` bookkeeping for syscall
### Memory
- userspace:
  - Addressing: `cr3` -> PGD(Page Global Directory) -> Page Upper Directory -> Page Middle Directory -> PTE (Page Table Entry) -> Offset
  - Process memory image loading: ELF Program header, GNU linker `ld`
  - Segments: `fork` (COW), `clone` for threading, `exit`, `mmap`, `shmat`, `brk`; `mm_struct -> vm_area_struct`, `mm_struct -> pgd`, `vm_file -> f_dentry -> d_inode -> i_mapping`, `page -> address_space -> inode`
- kernelspace:
  - Node, Zone, Page. Free page list, `vm_area_struct` list of a process, CPU cache, SMP.
  - MMU, TLB
  - kmalloc (directly mapping), vmalloc (non-continues), kmap (high-memory)
- Page faults:
  - Region valid, but page not allocated, Minor
  - Region not valid but is beside an expandable region like the stack, Minor
  - Page swapped out, in cache Minor, in storage Major
  - Page write when marked read-only, Minor if COW, SIGSEGV if bad write
  - Region is invalid or no permissions, Error, SIGSEGV
  - Fault occurred in the kernel portion address space, Minor
  - Fault occurred in the userspace region while in kernel mode, Error
- Memory ordering
  - Compile-time reordering: memory barrier `asm volatile("" ::: "memory")`
  - Runtime reordering:
### Process
#### Process states
- `ps` command to show state
  - `R` Running, `S` Interruptible sleep,  `D` Uninterruptible sleep, `T` Stopped, `Z` Zombie
- `task_struct -> state` in kernel `sched.h`
  - TASK_NEW, TASK_RUNNING (runnable), TASK_INTERRUPTIBLE(waiting for event), TASK_UNINTERRUPTIBLE(waken by kernel only), __TASK_STOPPED (by debugger), __TASK_TRACED(being ptraced), TASK_PARKED, TASK_DEAD,  tsk->exit_state EXIT_DEAD;EXIT_ZOMBIE;EXIT_TRACE,

#### Management
##### Creation
  - `fork` / `vfork`: a full copy and COW
  - `clone`: shared
#### Scheduler
  - the periodic scheduler (`scheduler_tick`)  and the main scheduler (`schedule()`) called directly in kernel or after returning from system calls.
  - Completely Fair Scheduling class
    - virtual clock, red-black tree

#### Resource Limits
- `rlimit` `/proc/self/limits`

### VFS
#### Key structs
- `superblock`: `dentry s_root` `s_inodes` `s_dirty` `file s_files` `block_device s_bdev`. For a particular file system.
- `vfsmount`: `superblock` dentry of mountpoint.
- `file`: `inode` `dentry`
- `dentry`: mapping cache from filename to `inode`, otherwise, searching from root.
- `inode`: `address_space`.
  - `inode` states:
    - Exists in memory but is not linked to any file and is not in active use.
    - In memory and being used for one or more tasks representing a file.
    - Its data contents have been changed, dirty.
- `task_struct`: `files_struct` -> `file` `fdtable`
- _page cache_ for memory mapping; _buffer cache_ the access units used are the individual blocks of a device and not whole pages.


### Container
#### Namespaces
- types: mnt, net, pid, user, ipc, cgroup, time, uts

#### Cgroups
- types: cpu,cpuacct,  memory, hugetlb,  net_cls,net_prio,  pids,  devices, blkio, systemd, perf_event, freezer, rdma

### Network
#### Protocols
##### TCP
  - Connection-oriented: 3 way handshake for establish, [SYNC:x]; [SYNC/ACK:y/x+1]; [ACK:y+1]. 4 way handshake for termination
  - In order: Sliding Window,
  - Streaming: MSS, URG/PUSH
  - Reliable: retransmission for missing segment (duplicate ACKs) or ACK-timeout, adaptive retransmission (RTT), flow control (AdvertisedWindowSize), congestion control (QoS, `CongestionWindow` > MSS)
  - Sliding Window Algorithm:
    - SeqNum, ACKNum, AdvertiseWindow.
    - `AdvertisedWindow = MaxRcvBuffer - ((NextByteExpected - 1) - LastByteRead)`
    - _Zero Window Probes_ when `AdvertisedWindow = 0`
    - Flags: UrgPrt, Push, Reset
  - _MSS Maximum Segment Size_ _MTU_` - Headers`
  - _Adaptive Retransmit_: Missing ACK, timeout, 2 estimated RTT
  - _Selective Acknowledge SACK_ cumulative acknowledgment with received not contiguous segments in TCP options

##### UDP
  - Message-oriented. IPv4 UDP header checksum is optional, IPv6 mandatory. UDP cork.
##### Stream Control Transmission Protocol (SCTP)
  - Message-oriented reliable stream
##### ICMP
  - RFC 792
  - Header: type, code, checksum, payload
  - Types: Echo Reply/Request; Destination Unreachable; Redirect Message; Router Advertisement/Router Solicitation; Time Exceeded; Parameter Problem;
##### MLPS
  - explicit routes using labels, a virtual circuit network from packet switch network.

##### IGMP
  - broadcast, self registered

##### Border Gateway Protocol (BGP)
  - autonomous systems, routing scalability, hierarchically aggregate routing information
  - border routers, TCP

#### Conntrack & NetFilter
  - Conntrack states: _NEW_, _ESTABLISHED_, _RELATED_, _INVALID_
  - NetFilter hooks: _PREROUTING_, _LOCAL_INPUT_, _FORWARD_, _LOCAL_OUTPUT_, _POSTROUTING_
  - Only the first package of a new connection will traverse the NetFilter table

#### Network virtualization
  - VETH: pair, receive immediately
  - VLAN: tag in L2 header, shared MAC between vlan device and its parent
  - XVLAN: XVLAN header in UDP
  - MACVLAN: MAC to namespace, unique MAC and IP for each device
  - IPVLAN: L2 same MAC diff IP; L3 mode router

#### Traffic control  
  - qdisc: egress package queue, pfifo_fast (single queue), mq (multi-queues), SFQ, Stochastic Fair Queuing, TBF, Token Bucket Filter, HTB, Hierarchical Token Bucket
  - classes: filters are attached
  - filter: package classification
  - policer
  - handler:

#### Network optimizations
  - RSS: Receive Side Scaling. NIC supported, NICs to different CPUs
  - RPS: Receive Packet Steering. Software RSS
  - RFS: Receive Flow Steering. to the CPU where the application thread consuming the packet

  - XPS: Transmit Packet Steering. To which queue, CPU map / receive queues map
  - TSO: TCP Segmentation Offload. single super frame into multiple frames
  - GSO: Generic Segmentation Offload. segmentation just before driver's xmit queue
  - GRO: Generic Receive Offload. combining “similar enough” packets together to reduce CPU. any frame assembled by GRO should be segmented to create an identical sequence of frames using GSO

#### Congestion control
  - Connectionless Flows, soft state for flow
  - Resource Allocations: Router-Centric versus Host-Centric; Reservation-Based versus Feedback-Based; Window Based versus Rate Based; Fairness
  - Queuing Disciplines: FIFO with tail drop, Fair Queuing (multi-queues & Round Robin).
  - Linux
    - default pfifo_fast
    - setting priorities: `setsockopt` with `SO_PRIORITY`; `tc filter`; `QoS` in package
    - bfifo, pfifo, Stochastic Fair Queuing(hash bucket), Hierarchical Token Bucket (tokens as credits)
  - TCP
    - Additive Increase/Multiplicative Decrease: timeout `CongestionWindow =/ 2`; ACK `CongestionWindow = MSS * (MSS / CongestionWindow)`
    - Slow Start: double `CongestionWindow` each _RTT_ until there is a loss
    - `CongestionThreshold`: `CongestionWindow` value that results from multiplicative decrease. After reaching `CongestionThreshold`, `CongestionWindow` adds one package per _RTT_
    - Quick Start: A Request rate in IP optioin of _SYNC_ package, if accpetable by routers.
    - Fast Retransmit: 3 duplicate ACKs before retransmit. Possible long delay of Timeout-retranmit
    - TCP CUBIC: large `latency*bandwidth` aka _long-fat_; regular interval of last congestion; a cubic function to adjust `CongestionWindow`; start fast, slow close to previous `CongestionWindow` then fast grow afterword.
    - Congestion Avoidance
      - DECbit in IP header (added by router, returned by RX back to TX)
      - Random Early Dectection: router drops a package before potential congestion,
      - Explicit Congestion Notification (ECN): Router notifies congestion in IP TOS (#1 bit ECN capable, #2 Congestion encountered), instead of package dropping (TX timeout or dup ACKs); `ECN-Echo` and `Congestion-Window-Reduced` in TPC header
    - QoS
      - Package classifying & scheduling

#### SDN
    - Google B4
#### Key data struct
  - net_device: ` ->net_device_ops -> ndo_start_xmit ` `->netdev_rx_queue ` ` ->netdev_queue `
  - sk_buff: ` ->net_device ` `->sock ` head, data, tail, end
  - proto: protocol specific ops `sendmsg sendpage`
  - sock / sock_common: `sk_buff_head	sk_receive_queue` `sk_buff_head	sk_write_queue ` `dst_entry`
  - socket: state, type, sock, proto_ops

#### Network stack

##### ingress
  - NIC & IRQ & DMA
  - NAPI: poll in driver (reduce IRQs). Driver disable IRQ when harvest packages.
    - `ethtool -S devname`, `/sys/class/net/eth0/statistics`, `/proc/net/dev`

  - Driver in interrupt handler: `softnet_data -> poll_list`, `__raise_softirq_irqoff(NET_RX_SOFTIRQ)`, `net_rx_action`
  - `net_rx_action`: `sd->poll_list` `n->poll` (NAPI poll) `__raise_softirq_irqoff`
  - `netif_receive_skb` default, _RPS_ `process_backlog`:  `do_xdp_generic`   `deliver_skb -> packet_type ->func`
  - `ip_rcv`: 	`NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, ..., ip_rcv_finsih)`
  - `ip_rcv_finsih`: `ip_rcv_options` `ip_route_input_mc` `ip_route_input_slow -> fib_lookup`
  - `dst_input`: `ip_local_deliver` `ip_defrag` `NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN, ip_local_deliver_finish)`
  - `ip_local_deliver_finish`: `ipprot->handler`  `tcp_v4_rcv`/`udp_rcv`/`icmp_rcv`
  - `udp_rcv`: `__udp4_lib_lookup_skb (connection-less) ==> udp_unicast_rcv_skb  ==>  udp_queue_rcv_skb  ==>  udp_queue_rcv_one_skb  ==>  __udp_enqueue_schedule_skb ==>  __skb_queue_tail(sk->sk_receive_queue, skb)`

##### egress
  - `socket(AF_INET, SOCK_XXX, IPPROTO_XXX)`  `inetsw_array[] = {SOCK_STREAM IPPROTO_TCP inet_stream_ops, SOCK_DGRAM IPPROTO_UDP inet_dgram_ops}` `inet_dgram_ops = {inet_sendmsg, inet_recvmsg}` `udp_prot = {udp_sendmsg, udp_recvmsg}`
  - `sock->ops->sendmsg (inet_sendmsg) ==> sk->sk_prot->sendmsg (udp_sendmsg)`
  - `upd_sendmsg`: IP options from sock, multicast?, `flowi4_init_output if not rtable found` `ip_make_skb` `ip_append_data ==> udp_push_pending_frames ==> udp_send_skb`
  - `udp_send_skb`:  `ip_send_skb  ==>  ip_local_out/__ip_local_out` `nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT, dst_output)`  
  - `dst_output/skb_dst(skb)->output`: _protocol independent destination cache_ dst from rtable.
  - `ip_output`: `NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, ip_finish_output)`
  - `ip_finish_output`: `ip_fragment` `ip_finish_output2`
  - `ip_finish_output2`: `ip_neigh_for_gw` `neigh_output`
  - `neigh_output`: `neigh_hh_output` with `hh_cache` or `neighbour->output arp_hh_ops={neigh_resolve_output} neighbor system ARP`
  - `neigh_hh_output`: `dev_queue_xmit`
  - `dev_queue_xmit` _TC_: `sch_handle_egress tcf_classify` `netdev_pick_tx`  
  - `__dev_xmit_skb`: `qdisc_run_begin` `sch_direct_xmit __qdisc_run  qdisc_restart` `qdisc_run_end`
  - `qdisc_restart`: `dequeue_skb` `sch_direct_xmit  dev_hard_start_xmit   xmit_one  netdev_start_xmit`  
  - `netdev_start_xmit`:  `net_device_ops->ndo_start_xmit` device finishes xmit and raises an IRQ, `napi_schedule ==> net_rx_action` clean up tx/rx queues in driver.
#### eBPF & XDP
