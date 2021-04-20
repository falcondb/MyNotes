##trouble shooting methodology
###Investigation:
- Any possible relevant events, such as hardware, kernel, platform software upgrade, unexpected workload, network traffic congestion, operation errors.
- Environment: own data center, cloud, customer system, network (virtual, routing, tunneling), the runtime, bare metal, VM, container, hardware.
- Pattern: repeatable or one time, event period, one particular environment or almost everywhere.

###Trouble shouting strategy and Plan:
- The scope of the event: single machine or large scope
- The computer source: CPU, memory, network, storage, or configuration.
- The software level: user space or application, kernel, hardware


###Executions:
####Examination:
  - System: uname, kernal configuration file (/boot/config), sysctl, lsmod, bootparam
  - storage: mount, df, du, lsof, fuser, dumpe2fs
  - CPU: uptime, ps, top, pidstat,  /proc /sys
  - memory: sysctl, /proc/memoryinfo, free, numactl
    - `/sys/devices/system/node/` for status, `/sys/kernel/mm/` for the features.
  - network: make sure enter the right namespace, ip a:l:route, bridge/brctl, iptables (nftables), tc, ss, `/sys/class/net`, nc/socat, dig, ebtables (netfilter in bridge)
  - Cgroups: where, for what, for whom, current status
  - Namepces: `nsenter` `unshare` (`clone`),
####Monitoring
  - Utilization, Saturation, Error (dmesg, /var/log/, /sys/, journalctl),
  - sar, iostate, vmstate, mpstat, /proc /sys, perf ftrace eBPF,
  - process: prtstat, pidstat, pstree
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
  - Node, Zone, Page. Free page list, `vm_area_struct` list of a process
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
    - exists in memory but is not linked to any file and is not in active use.
    - is in memory and is being used for one or more tasks representing a file.
    - Its data contents have been changed, dirty.
- `task_struct`: `files_struct` -> `file` `fdtable`
- _page cache_ for memory mapping; _buffer cache_ the access units used are the individual blocks of a device and not whole pages.


### Container
#### Namespaces
- types: mnt, net, pid, user, ipc, cgroup, time, uts

#### Cgroups
- types: cpu,cpuacct,  memory, hugetlb,  net_cls,net_prio,  pids,  devices, blkio, systemd, perf_event, freezer, rdma

### Network

#### Conntrack & NetFilter
  - Conntrack states: _NEW_, _ESTABLISHED_, _RELATED_, _INVALID_
  - NetFilter hooks: _PREROUTING_, _LOCAL INPUT_, _FORWARD_, _LOCAL OUTPUT_, _POSTROUTING_
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
