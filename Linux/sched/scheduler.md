## Schedulers in Linux

### [CFS @linux/doc](https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt)
- `/proc/sys/kernel/sched_min_granularity_ns`

### [Fixing SCHED_IDLE](https://lwn.net/Articles/805317/)
- Scheduling classes
  - STOP (stop_task.c): no user task ever gets scheduled with it, SMP only, kthread named migration/N. used by kernel for task migration, CPU hotplug, RCU, ftrace, clock events...
  - DL (deadline.c): kernel ensures deadline-scheduling guarantees, if not, sched_setattr() fails with the error `EBUSY`
  - RT (rt.c): higher-priority tasks before lower-priority tasks. `SCHED_FIFO` `SCHED_RR`
  - FAIR (fair.c): `SCHED_NORMAL` `SCHED_BATCH` `SCHED_IDLE`.
    - `SCHED_NORMAL`
    - `SCHED_BATCH`: run uninterrupted for a period of time and hence are normally scheduled only after finishing all the SCHED_NORMAL activity
    - `SCHED_IDLE`: kthread `swapper/N`
  - IDLE (idle.c): doesn't manage any user tasks and so doesn't implement a policy

- Facebook patches [1](https://lore.kernel.org/lkml/20190408214539.2705660-1-songliubraving@fb.com/) [2](https://lore.kernel.org/lkml/2E7A1CDA-0384-474E-9011-5B209A1A58DF@fb.com/xxxxxxxxx)

### Load Balancing on SMP Systems
- A scheduling domain is a set of CPUs which share properties and scheduling policies, and which can be balanced against each other.

- Each scheduling domain has a balancing policy set which is valid on that level of the hierarchy. The policy parameters include how often attempts should be made to balance loads across the domain, how far the loads on the component processors are allowed to get out of sync before a balancing attempt is made, how long a process can sit idle before it is considered to no longer have any significant cache affinity.

- During active balancing, the kernel walks up the domain hierarchy, starting at the current CPU's domain, and checks each scheduling domain to see if it is due to be balanced, and initiates a balancing operation if so.

- A softirq `open_softirq(SCHED_SOFTIRQ, run_rebalance_domains)` performs the regular load balancing. It is triggered in `scheduler_tick`

- It would pull a task from an overloaded CPU to the current one to rebalance tasks but it would not push one.

- Idle balancing is invoked as soon as a CPU goes idle.

 - selecting a runqueue for a new task
  - `sched_exec` after `exec` syscall, using `SD_BALANCE_EXEC` in calling schedule class `select_task_rq`
  - `wake_up_new_task` for a just created task whoken up for the first time, using `SD_BALANCE_FORK`
  - `try_to_wake_up` using `SD_BALANCE_WAKE`


### [batch/idle priority scheduling, SCHED_BATCH](https://lwn.net/Articles/3866/)


### [Deadline scheduling part 1 — overview and theory](https://lwn.net/Articles/743740/)

### [Per-entity load tracking](https://lwn.net/Articles/531853/)

### [Schedulers: the plot thickens](https://lwn.net/Articles/230574/)

### [Scheduling domains](https://lwn.net/Articles/80911/)

### [Real-Time Linux Kernel Scheduler](https://www.linuxjournal.com/article/10165?page=0%2C4)

### [NO_HZ: Reducing Scheduling-Clock Ticks](https://www.kernel.org/doc/Documentation/timers/NO_HZ.txt)

### General issues
[kernel/timer.c design](https://lwn.net/Articles/156329/)

### Spin lock
[Ticket spinlocks](https://lwn.net/Articles/267968/)

### [Scheduler Soft Affinity](https://blogs.oracle.com/linux/post/soft-affinity-when-hard-partitioning-is-too-much)
- if each DB instance has memory pinned to one NUMA node, autonuma can migrate threads to their corresponding nodes when all instances are busy, thus automatically partitioning them. It does NOT show much benefit.
- numactl only restricts the initial memory allocation to a NUMA node and autonuma balancer is free to migrate them later


- Under the hood the scheduler will add an extra set, `cpu_preferred`, a subset of `cpu_allowed`.
 - In the first level of search, the scheduler chooses the last level cache (LLC) domain, which is typically a NUMA node. Here the scheduler will always use cpu_preferred to prune out remaining CPUs. Once LLC domain is selected, it will first search the cpu_preferred set and then (cpu_allowed - cpu_preferred) set to find an idle CPU and enqueue the thread.


### [Source code analysis of linux Scheduler](https://programmer.ink/think/source-code-analysis-of-linux-scheduler-overview.html)
*Very detailed, must read*
- events triggering schedule
  - reset()??!!
  - `schedule()`
  - returing to user space from syscall, exception or interrupt
  - When kernel preemption is enabled
    - When preempt_enable() is invoked in the context of system call or exception interrupt, the system will be scheduled only when it is called at the *last time* (preempt_enable())
    - In the interrupt context, when returning from the interrupt processing function to the preemptive context

    - When it is in the period of interruption (top & bottom), scheduling is forbidden by the system, and then scheduling is allowed again after interruption.
    - For exceptions, the system does not prohibit scheduling, that is, in the context of exceptions, the system is likely to schedule.


### [A diagram of the relationships between key data structures in scheduler](https://programmer.group/images/article/fb45e188959cfb01a15d3e582570a4d8.jpg)


### [Digging into the Linux scheduler](https://deepdives.medium.com/digging-into-linux-scheduler-47a32ad5a0a8)


### [Deep dive in the scheduler @Youtube](https://www.youtube.com/watch?v=1xhK0cH2Dkg&list=PLQSwn2RzFCCbs8jacOkx-SH---fsDNGie)

## Scheduler Tools
### [rt-app](https://github.com/scheduler-tools/rt-app)
