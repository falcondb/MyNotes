# Cgroups
[Control groups series by Neil Brow](https://lwn.net/Articles/604609/)

A list of the cgroup series
***
[Control groups, part 1: On the history of process grouping](https://lwn.net/Articles/603762/)

History of the concepts of process group, job, session.
The concept is still here in Linux
```
ps -axgo comm,sess,pgrp
```

[Control groups, part 2: On the different sorts of hierarchies](https://lwn.net/Articles/604413/)

[Control groups, part 3: First steps to control](https://lwn.net/Articles/605039/)

>net_cl and net_prio apply just to that cgroup and any sockets associated with processes in that cgroup. They do not automatically apply to sockets in processes in child cgroups.

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

RIP, Rami!

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
