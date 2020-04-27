# Cgroups
[Control groups series by Neil Brow](https://lwn.net/Articles/604609/)

A list of the cgroup series
***
[Control groups, part 1: On the history of process grouping](https://lwn.net/Articles/603762/)


***
[The unified control group hierarchy in 3.16](https://lwn.net/Articles/601840/)

[Cgroup talk by Rami Rosen](https://www.youtube.com/watch?v=zMJD8PJKoYQ). RIP, Rami!

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