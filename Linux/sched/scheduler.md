## Schedulers in Linux


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
A scheduling domain is a set of CPUs which share properties and scheduling policies, and which can be balanced against each other.

Each scheduling domain has a balancing policy set which is valid on that level of the hierarchy. The policy parameters include how often attempts should be made to balance loads across the domain, how far the loads on the component processors are allowed to get out of sync before a balancing attempt is made, how long a process can sit idle before it is considered to no longer have any significant cache affinity.



### [Deadline scheduling part 1 — overview and theory](https://lwn.net/Articles/743740/)

### [Per-entity load tracking](https://lwn.net/Articles/531853/)

### [NO_HZ: Reducing Scheduling-Clock Ticks](https://www.kernel.org/doc/Documentation/timers/NO_HZ.txt)

### General issues
[kernel/timer.c design](https://lwn.net/Articles/156329/)

### Spin lock
[Ticket spinlocks](https://lwn.net/Articles/267968/)


### [Digging into the Linux scheduler](https://deepdives.medium.com/digging-into-linux-scheduler-47a32ad5a0a8)


## Scheduler Tools
### [rt-app](https://github.com/scheduler-tools/rt-app)
