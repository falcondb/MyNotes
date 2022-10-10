# Cgroups
[Control groups series by Neil Brow](https://lwn.net/Articles/604609/)

A list of the cgroup series

[Control groups, part 1: On the history of process grouping](https://lwn.net/Articles/603762/)

History of the concepts of process group, job, session.
The concept is still here in Linux
```
ps -axgo comm,sess,pgrp
```

[Control groups, part 2: On the different sorts of hierarchies](https://lwn.net/Articles/604413/)

[Control groups, part 3: First steps to control](https://lwn.net/Articles/605039/)

>net_cl and net_prio apply just to that cgroup and any sockets associated with processes in that cgroup. They do not automatically apply to sockets in processes in child cgroups.

[Network classifier cgroup, net_cls](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/net_cls.html)
```
mount -t cgroup -o net_cls net_cls /sys/fs/cgroup/net_cls
echo 0x100001 >  /sys/fs/cgroup/net_cls/0/net_cls.classid

tc qdisc add dev eth0 root handle 10: htb
tc class add dev eth0 parent 10: classid 10:1 htb rate 40mbit
tc filter add dev eth0 parent 10: protocol ip prio 10 handle 1: cgroup
iptables -A OUTPUT -m cgroup ! --cgroup 0x100001 -j DROP
```

[Network priority cgroup, net_prio](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/net_prio.html)
ominally, an application would set the priority of its traffic via the SO_PRIORITY socket option, not always possible.
_net_prio.prioidx_: This file is read-only, and is simply informative. It contains a unique integer value that the kernel uses as an internal representation of this cgroup.
_net_prio.ifpriomap_: This file contains a map of the priorities assigned to traffic originating from processes in this group and egressing the system on various interfaces. It contains a list of tuples in the form `ifname priority`.
```
mount -t cgroup -onet_prio none /sys/fs/cgroup/net_prio
echo "eth0 5" > /sys/fs/cgroups/net_prio/iscsi/net_prio.ifpriomap
```
This command would force any traffic originating from processes belonging to the iscsi net_prio cgroup and egressing on interface eth0 to have the priority of said traffic set to the value 5.

Priorities are set immediately prior to queueing a frame to the device queueing discipline, so priorities will be assigned prior to the hardware queue selection being made.

[TC cgroup manual](http://manpages.ubuntu.com/manpages/xenial/man8/tc-cgroup.8.html)

>The devices subsystem can either allow or deny all accesses by default, and then have a list of exceptions where access is denied or allowed, respectively.

>In freezer subsystem, the whole group of processes from a subgroup is stopped or restarted

>The perf_event cgroup subsystem interprets "within" in a fully hierarchical sense is based on an ID number that may be shared by some but not all groups in a subtree. walks up the tree from a process to find the answer.

>The cpuset cgroup subsystem identifies a set of processors on which each process in a group may run, and it also identifies a set of memory nodes from which processes in a group can allocate memory.
>
>The cpuset propagates any changes made to a parent into all descendants when appropriate.

[Control groups, part 4: On accounting](https://lwn.net/Articles/606004/)

worth of reading it more, a very deep discussion

>The memory cgroup subsystem allocates three res_counters, one for user-process memory usage, one for the sum of memory and swap usage, and one for usage of memory by the kernel on behalf of the process. Together with the one res_counter allocated by hugetlbe.

>When one of the various memory resources is requested by a process, the res_counter code will walk up the parent pointers, checking if limits are reached and updating the usage at each ancestor.

>The memory controller will request that a full 32 be approved by the res_counter

[Control groups, part 5: The cgroup hierarchy](https://lwn.net/Articles/606699/)

>To understand it we need at least a brief introduction to Network Traffic Control (NTC). The NTC mechanism is managed by the tc program. This tool allows a "queueing discipline" (or "qdisc") to be attached to each network interface. Some qdiscs are "classful" and these can have other qdiscs attached beneath them, one for each "class" of packet. If any of these secondary qdiscs are also classful, a further level is possible, and so on. This implies that there can be a hierarchy of qdiscs, or several hierarchies, one for each network interface.

[Control groups, part 6: A look under the hood](https://lwn.net/Articles/606925/)

Deep discussions about the implementation of cgroup hierarchy with processes (thrend groups).

How to reduce the combinatorial explosion of cgroup hierarchy and processes.

The locking on the processes (thrend groups) when updating their cgroups

Cgroup linkage to reduce the combinatorial explosion

>We had a reminder that terminology can be challenging, and that we should be careful when interpreting what we read in the kernel code

>Locking can be tricky. It is generally best to avoid requiring locks where possible, and doing so is easier when the data structures are simple and elegant.

>the combinatorial explosion we were concerned about is unavoidable. cgroup_cset_links are created automatically, rather than requiring mkdir requests, so possibly having a proliferation of links is cheaper than a proliferation of cgroups. Rather than being concerned about a proliferation, we could instead be concerned about the size of cgroups and what mechanisms could be used to create cgroups automatically.

***

[CFS group scheduling](https://lwn.net/Articles/240474/)

>These patches turn a scheduling entity into a hierarchical structure, Each scheduling entity of this type has its own run queue within it

>When the scheduler goes to pick the next task to run, it looks at all of the top-level scheduling entities and takes the one which is considered most deserving of the CPU. If that entity is not a process, then the scheduler looks at the run queue contained within that entity and starts over again. Things continue down the hierarchy until an actual process is found, at which point it is run.

>So any particular policy can be implemented through the creation of a simple, user-space daemon which responds to process creation events by placing newly-created processes in the right group.

[CFS bandwidth control](https://lwn.net/Articles/428230/)

>The scheduler will not, however, insist on equal utilization when there is free CPU time available; rather than let the CPU go idle, the scheduler will give any left-over time to processes which can make use of it

>cpu.cfs_period_us defines the period over which the group's CPU usage is to be regulated, and cpu.cfs_quota_us controls how much CPU time is available to the group over that period. (see Redhat doc, more clear)



[Schedulers: the plot thickens](https://lwn.net/Articles/230574/)

[The unified control group hierarchy in 3.16](https://lwn.net/Articles/601840/)


>The unified hierarchy can be instantiated alongside older hierarchies, but controllers cannot be shared between the unified hierarchy and any others.

>In Cgroup v1, controllers are attached to control groups by specifying options to the mount command that creates the hierarchy. In the unified hierarchy world, instead, all controllers are [Control group hierarchy]attached to the root of the hierarchy

>cgroup.controllers that lists the controllers that can be enabled for children of that group
>cgroup.subtree_control, lists the controllers that are actually enabled

>A controller can only be enabled in a specific control group if it is enabled in that group's parent; it is possible to disable a controller at a lower level

>the cgroup.subtree_control file can only be used to change the set of active controllers if the associated group contains no processes

>cgroup.populated; reading it will return a nonzero value if there are any processes in the group (or its descendants). By using poll() on this file, a process can be notified if a control group becomes completely empty; the process would presumably respond by cleaning up and removing the group

[unified-hierarchy.txt](https://lwn.net/Articles/601923/)

```
mount -t cgroup -o __DEVEL__sane_behavior cgroup $MOUNT_POINT
```
>All controllers which are not bound to other hierarchies are automatically bound to unified hierarchy and show up at the root of it

>cgroup.subtree_control, All cgroups on unified hierarchy have a "cgroup.subtree_control" file, which governs which controllers are enabled on the children of the cgroup.
>When read, the file contains a space-separated list of currently enabled controllers.  A write to the file should contain a space-separated list of controllers with '+' or '-' prefixed

>cgroup.controllers, Read-only "cgroup.controllers" file contains a space-separated list of controllers which can be enabled in the cgroup's "cgroup.subtree_control" file
>>In the root cgroup, this lists controllers which are not bound to other hierarchies and the content changes as controllers are bound to and unbound from other hierarchies.
>>In non-root cgroups, the content of this file equals that of the parent's "cgroup.subtree_control" file as only controllers enabled from the parent can be used in its children.

>Except for the root, only cgroups which don't contain any task may have controllers enabled in their "cgroup.subtree_control" files.


[Cgroup talk by Rami Rosen](https://www.youtube.com/watch?v=zMJD8PJKoYQ)

Read [Resource Groups](https://lwn.net/Articles/679940/)! Seems a new generation after Cgroups

```
mount -t cgroup2 none $MOUNT_POINT  #Mount Cgroup V2
```

>Creation of new subgroups in cgroups v2 is done with mkdir groupName, and removal is done with rmdir groupName.

>A cgroup root object is created, with three cgroup core interface files beneath it:
>>cgroup.controllers

>>cgroup.procs (put PIDs in);

>>cgroup.subtree_control
```
echo "+memory" > /sys/fs/cgroup2/cgroup.subtree_control
echo "-memory" > /sys/fs/cgroup2/cgroup.subtree_control
```

>>cgroup.events: reflects the number of processes attached to the subgroup: 0 when there are no processes attached to that subgroup or its descendants; 1 when there are one or more processes attached

>enable the controller in the direct child subgroups and no other descendants

>Unlike in cgroups v1, in cgroups v2 you can attach processes only to leaves

>In cgroups v1, a process can belong to many subgroups, if those subgroups are in different hierarchies with different controllers attached. But, because belonging to more than one subgroup made it difficult to disambiguate subgroup membership, in cgroups v2, a process can belong only to a single subgroup.

***
[Tejun Heo on cgroups and cgroups v2](https://www.youtube.com/watch?v=PzpG40WiEfM)

[Thread-level management in control groups](https://lwn.net/Articles/656115/) [The evolution of control groups](https://lwn.net/Articles/571977/)
>A few users of thread-level cgroup control surfaced in the ensuing discussion; the most vocal of them was Paul Turner of Google, who asserted that this ability is an important part of how systems are managed there. One use case mentioned was the division of a job into work and support threads. The threads doing the "real work" should get the bulk of the available CPU time, but an application will typically want to guarantee a minimum of time to the support threads as well. Putting the two types of threads into different control groups allows this policy to be implemented in a fairly straightforward way.

>The issue, they said, is fundamental to the design of the subsystem, and it is not reasonable to expect that a solu

>Of all the controllers only the CPU controller has any business working with individual threads. ... Linus wondered if it was really true that only the CPU controller needs to look at individual threads; some server users, he said have wanted per-thread control for other resources as well.


[Fixing control groups](https://lwn.net/Articles/484251/)

[A control group manager](https://lwn.net/Articles/618411/)

[Resource Groups: Thread-level control with resource groups](https://lwn.net/Articles/679940/)

[cpuset.rst](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/cgroup-v1/cpusets.rst)
READ it first every time working with cpuset!!!!
> Cpusets provide a mechanism for assigning a set of CPUs and Memory Nodes to a set of tasks. In this document "Memory Node" refers to an on-line node that contains memory.

> Requests by a task, using the sched_setaffinity(2) system call to include CPUs in its CPU affinity mask, and using the mbind(2) and set_mempolicy(2) system calls to include Memory Nodes in its memory policy, are both filtered through that task's cpuset, filtering out any CPUs or Memory Nodes not in that cpuset

> The management of large computer systems, with many processors (CPUs), complex memory cache hierarchies and multiple Memory Nodes having non-uniform access times (NUMA) presents additional challenges for the efficient scheduling and memory placement of processes. But larger systems, which benefit more from careful processor and memory placement to reduce memory access times and contention, and which typically represent a larger investment for the customer, can benefit from explicitly placing jobs on properly sized subsets of the system.

> You should mount the "cgroup" filesystem type in order to enable browsing and modifying the cpusets presently known to the kernel. No new system calls are added for cpusets - all support for querying and modifying cpusets is via this cpuset file system

> If a cpuset is cpu or mem exclusive, no other cpuset, other than a direct ancestor or descendant, may share any of the same CPUs or Memory Nodes.

> The memory_pressure of a cpuset provides a simple per-cpuset metric of the rate that the tasks in a cpuset are attempting to free up in use memory on the nodes of the cpuset to satisfy additional memory requests.

> There are two boolean flag files per cpuset that control where the kernel allocates pages for the file system buffers and related in kernel data structures. They are called 'cpuset.memory_spread_page' and 'cpuset.memory_spread_slab'.
> If the per-cpuset boolean flag file 'cpuset.memory_spread_page' is set, then the kernel will spread the file system buffers (page cache) evenly over all the nodes that the faulting task is allowed to use, instead of preferring to put those pages on the node where the task is running.
> If the per-cpuset boolean flag file 'cpuset.memory_spread_slab' is set, then the kernel will spread some file system related slab caches, such as for inodes and dentries evenly over all the nodes that the faulting task is allowed to use, instead of preferring to put those pages on the node where the task is running.
> Setting memory spreading causes allocations for the affected page or slab caches to ignore the task's NUMA mempolicy and be spread instead.


> The algorithmic cost of load balancing and its impact on key shared kernel data structures such as the task list increases more than linearly with the number of CPUs being balanced. So the scheduler has support to partition the systems CPUs into a number of sched domains such that it only load balances within each sched domain

> The algorithmic cost of load balancing and its impact on key shared kernel data structures such as the task list increases more than linearly with the number of CPUs being balanced. So the scheduler has support to partition the systems CPUs into a number of sched domains such that it only load balances within each sched domain
> If a task is moved from one cpuset to another, then the kernel will adjust the task's memory placement, as above, the next time that the kernel attempts to allocate a page of memory for that task.

> If a cpuset has its 'cpuset.cpus' modified, then each task in that cpuset will have its allowed CPU placement changed immediately. Similarly, if a task's pid is written to another cpuset's 'tasks' file, then its allowed CPU placement is changed immediately.

> In summary, the memory placement of a task whose cpuset is changed is updated by the kernel, on the next allocation of a page for that task, and the processor placement is updated immediately.


[Freezer Cgroup @kernel doc](https://www.kernel.org/doc/Documentation/cgroup-v1/freezer-subsystem.txt)

> The freezer allows the checkpoint code to obtain a consistent image of the tasks by attempting to force the tasks in a cgroup into a quiescent state. Once the tasks are quiescent another task can walk /proc or invoke a kernel interface to gather information about the quiesced tasks. This also allows the checkpointed tasks to be migrated between nodes

> Sequences of SIGSTOP and SIGCONT are not always sufficient for stopping and resuming tasks in userspace. Both of these signals are observable from within the tasks

> While SIGSTOP cannot be caught, blocked, or ignored it can be seen by waiting or ptracing parent tasks. Any programs designed to watch for SIGSTOP and SIGCONT could be broken by attempting to use SIGSTOP and SIGCONT to stop and resume tasks.

- freezer.state: effective state
- freezer.self_freezing: THAWED or FROZEN
- freezer.parent_freezing:

```
# mkdir /sys/fs/cgroup/freezer
# mount -t cgroup -ofreezer freezer /sys/fs/cgroup/freezer
# mkdir /sys/fs/cgroup/freezer/0
# echo $some_pid > /sys/fs/cgroup/freezer/0/tasks
```
