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
- `task_struct`: `files_struct` -> `file`
- _page cache_ for memory mapping; _buffer cache_ the access units used are the individual blocks of a device and not whole pages.


### Container
#### Namespaces
- types: mnt, net, pid, user, ipc, cgroup, time, uts

#### Cgroups
- types: cpu,cpuacct,  memory, net_cls,net_prio,  pids,  devices, systemd, perf_event, hugetlb, freezer, blkio, rdma
