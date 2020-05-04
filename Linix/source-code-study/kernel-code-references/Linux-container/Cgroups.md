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
[ejun Heo on cgroups and cgroups v2](https://www.youtube.com/watch?v=PzpG40WiEfM)

[Fixing control groups](https://lwn.net/Articles/484251/)

[A control group manager](https://lwn.net/Articles/618411/)

[Resource Groups: Thread-level control with resource groups](https://lwn.net/Articles/679940/)