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
  - core (/proc/sys/net/core)
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
  - Addressing: `cr3` (x86 TLB flushing) -> PGD(Page Global Directory) -> Page Upper Directory -> Page Middle Directory -> PTE (Page Table Entry) -> Offset
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

### Container
#### Namespaces
  - types: mnt, net, pid, user, ipc, cgroup, time, uts
  - commands: `nsenter -t $PID $NS` `unshare $NS`
  - system calls: `clone() setns() unshare()`
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
      - Random Early Detection: router drops a package before potential congestion,
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
  - `udp_sendmsg`: IP options from sock, multicast? if `sock->dst_entry`? use it: `flowi4_init_output; ip_route_output_flow ==> __ip_route_output_key/ip_route_output_key_hash` if not rtable found `ip_make_skb` `ip_append_data ==> udp_push_pending_frames ==> ip_finish_skb; udp_send_skb`
  - `udp_send_skb`:  `ip_send_skb  ==>  ip_local_out/__ip_local_out` `nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT, dst_output)`  
  - `dst_output/skb_dst(skb)->output`: _protocol independent destination cache_ dst from rtable.
  - `ip_output`: `NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, ip_finish_output)`
  - `ip_finish_output`: `ip_fragment` `ip_finish_output2`
  - `ip_finish_output2`: `ip_neigh_for_gw` `neigh_output`
  - `neigh_output`: `neigh_hh_output` with `hh_cache` or `neighbour->output arp_hh_ops={neigh_resolve_output} neighbor system ARP`
  - `neigh_hh_output`: `dev_queue_xmit`
  - `dev_queue_xmit` _TC_: `sch_handle_egress tcf_classify` `netdev_pick_tx`  
  - `__dev_xmit_skb`: `qdisc_run_begin` (q is running) `sch_direct_xmit` (hardware is not available or busy) `__qdisc_run  qdisc_restart` `qdisc_run_end`
  - `qdisc_restart`: `dequeue_skb` `sch_direct_xmit  dev_hard_start_xmit   xmit_one(onl on skb)  netdev_start_xmit`  
  - `netdev_start_xmit`:  `net_device_ops->ndo_start_xmit` device finishes xmit and raises an IRQ, `napi_schedule ==> net_rx_action` clean up tx/rx queues in driver.

#### eBPF & XDP
  - System call: `int bpf(int cmd, union bpf_attr *attr, unsigned int size)`
  - Registers:
    - `r10` frame pointer
    - `r0` bpf return value / function return value
    - `r1` context argument of the BPF program
    - `r1 - r5` arguments of the BPF helper `r6 - r9` reserve for helper
  - 1M instruction limit
  - BPF instruction
    - 64 bit --- op:8, dst_reg:4, src_reg:4, off:16, imm:32
    - op:8 --- code:4, source:1 and class:3
  - Map
    - data sharing between BPF programs, or kernal and userspace through fd
    - in kernel, map is stored in file's private_data
  - Object Pinning: BPF program or map at `/sys/fs/bpf/`
  - Hardening:
    - BPF interpreter image and JIT compiled image are locked as read-only pages
    - retpoline between BPF calls
    - constant blinding: `rnd ^ imm  ==>  ^rnd`
#### Memory Management
##### Memory hierarchy
  - NUMA `pglist_data`:
    - `zone` _ZONE HIGHMEM_ (not directly mapped to kernel address), _ZONE NORMAL_ (mapped to upper linear address), _ZONE DMA / ZONE_DMA32_ (ISA devices, architecture dependent)
      - `struct free_area free_area[MAX_ORDER]` free areas of different sizes
    - `struct page *node_mem_map`
      - `struct list_head lru`
			-	`struct address_space *mapping` `inode` associated
    - `struct per_cpu_pageset __percpu *pageset` _Slab_

  - `struct task_struct ` -> `struct mm_struct`
    - `struct vm_area_struct *mmap` mmap of the process
      - `struct file * vm_file` -> `struct inode` -> `struct address_space` -> `struct rb_root_cached	i_mmap`
    - `pgd_t * pgd` PGD to PTE
    - Red-Black tree for searching `struct vm_area_struct`


#### Virtual File System
##### superblock
  - block size, max file, size,
  - inode operation functions
  - dirty inodes in superblock
  - `struct dentry		*s_root`, `struct list_head	s_mounts`
  - `struct hlist_node	s_instances`

##### inode
  - Metadata & Data segment (file context)
  - Entries in directory inode: inode number; file/directory name
  - Softlink: different inodes; Hardlink: same inode
  - per cpu counts: nr_inodes; dirty inodes in superblock
  - `struct address_space	*i_mapping`

##### files_struct
  - `struct file __rcu * fd_array[]` opened files
  - `struct fdtable` using RCU to update the changes in fds

##### file
  - `struct path f_path` `struct path {struct vfsmount *mnt; struct dentry *dentry }`
  - `struct inode		*f_inode`
  - `struct address_space *f_mapping`

##### dentry
  - link between file name and inode
  - represents a file or a directory
  - `struct inode *d_inode	struct qstr d_name` `struct dentry *d_parent,  d_child, d_subdirs`

##### address space
  - link between page cache and data resource
  - `struct inode *host`
  - `struct xarray		i_pages`
  - Radix tree for the pages

##### vfsmount
  - `struct vfsmount *mnt_parent`
  - `struct dentry *mnt_mountpoint`
  - `struct dentry *mnt_root` pointer to superblock
  - `struct super_block *mnt_sb`

##### Operations
  - super_operations: inode ops, such as `alloc_inode, write_inode, dirty_inode ...`; fs ops `sync_fs, freeze_fs ...`
  - inode_operations
  - file_operations: open, read, write, mmap, fsync, flush
  - dentry_operations
  - address_space_operations: `readpage, writepage, set_page_dirty`

#### Cache
  - _Page cache_ cache operations in units of page (page size), mmap
  - _Buffer cache_ cache operations in units of device block (block size). `struct buffer_head`. `struct page -> prvate` links Page and Buffer cache


## Instruction set architecture
### x86-64  
  - Registers: 16 * 64-bit general purpose registers; 16 * 128-bit SSE (Streaming SIMD Extensions) registers; 8 * 80b-it floating registers
  - Stack frame:
    - 0(%rbp):previous %rbp of the caller; 8(%rbp): return address; 8n+16(%rbp): function arugments in 8 bytes
    - -8(%rbp) --> 0(%rsp): local variables
  - Register usage:
    - `rax`: 1st return register;  `rbp`: caller-saved; `rcp`: 4th integer argument; `rdx`: 3rd integer argument / 2nd return register; `rsi rdi r8 r9` 2nd 1st 5th 6th integer argument; `r15` _GOT_ base pointer
  - Address space
      - 48 bit to 0x0000 7fff ffff ffff
      - Text segment from 0x40 0000; Stack segment downward from 0x800 0000 0000
  - _rFLAGS_: `CF` no carry; `ZF` No zero result; `OF` no overflow    

### Arm
#### Arm32
  - Registers
    - 16 * 32-bit general purpose integer registers
  - Register usage:
    - `X0-X3` Parameter `X0 X1` Result Registers
    - `X11` Frame base
    - `X12` used by linkers to insert veneers
    - `X13` Stack pointer
    - `X14` _link register_
    - `X15` Program counter _PC_
#### Arm64
  - Registers
    - 31 * 64-bit general purpose registers
    - `XZR WZR` zero registers
    - `sp` stack pointer, `PC` program counter (not general purpose registers)
    - `MSR` instruction to read/write _system registers_ from/to general purpose register
  - Register usage:
      - `X0-X7` Parameter and `X0 X1` Result Registers.
      - `XR (X8)` to the memory allocated by the caller for returning the struct.
      - `X9-X15` Corruptible Registers
      - `IP0 (X16)` `IP1 (X17)` used by linkers to insert veneers between the caller and callee
      - `X19-X28` Callee-saved Registers
      - `FP (X29)` Frame
      - `LR (X30)` _link register_ for function return addr
      - `TPIDRRO_EL0` CPU number
      - The program counter and the stack pointer aren't indexed registers
  - Pipeline: IF, ID are in order execution; EX, MEM, WB are out of order execution
  - System call:
    - `SVC` Supervisor call: Used by an application to call the OS
    - `HVC` Hypervisor call: Used by an OS to call the hypervisor
    - `SMC` Secure monitor call: Used by an OS or hypervisor to call the EL3 firmware
  - Endian: most of the Arm implements use little endian  
## Linker, Loader & ELF
  - _GOT_ entry: symbol: address after dynamic loading
  - _GOT.PLT_ entry: function name: address after dynamic loading. The first hit, the address is jumping back to GOT.PLT, then running loader to resolve the function's virtual address
