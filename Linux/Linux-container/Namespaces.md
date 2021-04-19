# Namespace
***
[Namespaces in operation, part 1: namespaces overview](https://lwn.net/Articles/531114/#series_index)
Functionally completed at version 3.8 (2013).
namespace-related APIs: clone(), unshare(), and setns().
Mount namespaces isolate the set of filesystem mount points seen by a group of processes

[Namespaces in operation, part 2: the namespaces API](https://lwn.net/Articles/531381/)
clone API;
/proc/PID/ns files, the Magic IDs in ns files are INode IDs;
setns API: the fd is the one by open the /proc/PID/ns files;

[Namespaces in operation, part 3: PID namespaces](https://lwn.net/Articles/531419/)
the procfs needs to be mounted from within that PID namespace
```
# mount -t proc proc /mount_point
mount("proc", mount_point, "proc", 0, NULL);
```

[Namespaces in operation, part 4: more on PID namespaces](https://lwn.net/Articles/532748/)
>The first process created inside a PID namespace gets a process ID of 1 within the namespace. This process has a similar role to the init process on traditional Linux systems. In particular, the init process can perform initializations required for the PID namespace as whole and becomes the parent for processes in the namespace that become orphaned.
The ppid of an orphan process is the init process with pid = 1

>Other processes in the namespace (even privileged processes) can send only those signals for which the init process has established a handler.

>Signals can also be sent to the PID namespace init process by processes in ancestor PID namespaces. Again, *only the signals for which the init process has established a handler can be sent, with two exceptions: _SIGKILL_ and _SIGSTOP_. When a process in an ancestor PID namespace sends these two signals to the init process, they are forcibly delivered (and can't be caught). The SIGSTOP signal stops the init process; SIGKILL terminates it. Since the init process is essential to the functioning of the PID namespace, *if the init process is terminated by SIGKILL (or it terminates for any other reason), the kernel terminates all other processes in the namespace by sending them a SIGKILL signal*.

>Normally, a PID namespace will also be destroyed when its init process terminates. However, there is an unusual corner case: the namespace won't be destroyed as long as a /proc/PID/ns/pid file for one of the processes in that namespaces is bind mounted or held open.

An example and code to show the orphan and its parent process when the orphan terminates

[Namespaces in operation, part 5: User namespaces](https://lwn.net/Articles/532593/)
>When a user namespace is created, the first process in the namespace is granted a full set of capabilities in the namespace
>If a user ID has no mapping inside the namespace, then system calls that return user IDs return the value defined in the file /proc/sys/kernel/overflowuid
>Although the new process has a full set of capabilities in the new user namespace, it has no capabilities in the parent namespace.

Mapping user and group IDs
```
/proc/PID/uid_map       #ID-inside-ns   ID-outside-ns   length
/proc/PID/gid_map       #ID-inside-ns   ID-outside-ns   length
echo '0 1000 1' > /proc/xxxx/gid_map
```
>Defining a mapping is a one-time operation per namespace: we can perform only a single write to a uid_map file of exactly one of the processes in the user namespace.
>The number of lines that may be written to the file is currently limited to five.
>The /proc/PID/uid_map file is owned by the user ID (with CAP_SETUID) that created the namespace, and is writeable only by that user (or a privileged user)

Capabilities, execve(), and user ID 0
>when a process with non-zero user IDs performs an execve(), the process's capability sets are cleared. it is necessary to create a user ID mapping inside the user namespace.


[Namespaces in operation, part 6: more on user namespaces](https://lwn.net/Articles/540087/)
>A process can change its user-namespace membership using setns(), if it has the CAP_SYS_ADMIN capability in the target namespace; in that case, it obtains a full set of capabilities upon entering the target namespace.

>a clone(CLONE_NEWUSER) call creates a new user namespace and places the new child process in that namespace; unshare() places the caller in the new user namespace, and the parent of that namespace is the caller's previous user namespace

>whether or not a process has capabilities in a particular user namespace depends on its namespace membership and the parental relationship between user namespaces. The rules are as follows:
>>A process has a capability inside a user namespace if it is a member of the namespace and that capability is present in its effective capability set.

>>If a process has a capability in a user namespace, then it has that capability in all child (and further removed descendant) namespaces as well

>>When a user namespace is created, the kernel records the effective user ID of the creating process as being the "owner" of the namespace. A process whose effective user ID matches that of the owner of a user namespace and which is a member of the parent namespace has all capabilities in the namespace.

>the kernel guarantees that the CLONE_NEWUSER flag is acted upon first, creating a new user namespace in which the to-be-created child has all capabilities. The kernel then acts on all of the remaining CLONE_NEW* flags, creating corresponding new namespaces and making the child a member of all of those namespaces.

A deep discussion about the Capabilities in user Namespaces


[Namespaces in operation, part 7: Network namespaces](https://lwn.net/Articles/580893/)
Create a new Nerwork namespace by clone() or cmdline like
```
ip netns add netns1
```
When the ip tool creates a network namespace, it will create a bind mount for it under ```/var/run/netns```

There are several ways to connect the namespace to the internet if that is desired. A bridge can be created in the root namespace and the veth device from netns1. Alternatively, IP forwarding coupled with network address translation (NAT) could be configured in the root namespace.

Non-root processes that are assigned to a namespace only have access to the networking devices and configuration that have been set up in that namespace—root can add new devices and configure them.



[Linux Man IP netns](http://man7.org/linux/man-pages/man8/ip-netns.8.html) for more details

>New network namespaces will have a loopback device but no other network devices. Aside from the loopback device, each network device can only be present in a single network namespace.

>Physical devices cannot be assigned to namespaces other than the root.

>Virtual network devices can be created and assigned to a namespace.

virtual ethernet devices need to be created and configured:
```
ip link add veth0 type veth peer name veth1
ip link set veth1 netns netns1
ip netns exec netns1 ifconfig veth1 10.1.1.1/24 up
ifconfig veth0 10.1.1.2/24 up
```

> A bridge can be created in the root namespace and the veth device from netns1. Alternatively, IP forwarding coupled with network address translation (NAT) could be configured in the root namespace


[Mount namespaces and shared subtrees](https://lwn.net/Articles/689856/)
>When a new mount namespace is created, it receives a copy of the mount point list replicated from the namespace of the caller of clone() or unshare().
>Mount points can be independently added and removed in each namespace

>[Shared subtrees](https://lwn.net/Articles/159077/)
>>Under the shared subtrees feature, each mount point is marked with a "propagation type", which determines whether mount points created and removed under this mount point are propagated to other mount points. There are four different propagation types:
>>>MS_SHARED: This mount point shares mount and unmount events with other mount points that are members of its "peer group" propagate to the peer group, so that the mount or unmount will also take place under each of the peer mount points.

>>>MS_PRIVATE: This is the converse of a shared mount point. The mount point does not propagate events to any peers, and does not receive propagation events from any peers.

>>>MS_SLAVE: A slave mount has a master—a shared peer group whose members propagate mount and unmount events to the slave mount. However, the slave mount does not propagate events to the master peer group.

>>>MS_UNBINDABLE: This mount point is unbindable. Like a private mount point, this mount point does not propagate events to or from peers. In addition, this mount point can't be the source for a bind mount operation.

>the propagation type is a per-mount-point setting

>the propagation type determines the propagation of mount and unmount events immediately under the mount point

A *peer group* is a set of mount points that propagate mount and unmount events to one another.

```
/proc/PID/mountinfo
```

[Mount namespaces, mount propagation, and unbindable mounts](https://lwn.net/Articles/690679/)

***
[User namespaces progress](https://lwn.net/Articles/528078/)


***
[LCE: The failure of operating systems and how we can fix it](https://lwn.net/Articles/524952/)

>Hypervisors
>>Glauber chose KVM. Under KVM, the Linux kernel is itself the hypervisor. That makes sense, Glauber said, because all of the resource isolation that should be done by the hypervisor is already done by the operating system. The hypervisor has a scheduler, as does the kernel. So the idea of KVM is to simply re-use the Linux kernel's scheduler to schedule virtual machines. The hypervisor has to manage memory, as does the kernel, and so on; everything that a hypervisor does is also part of the kernel's duties.

>Containers
>>Hypervisors handle these use cases by running multiple kernel instances. But, he asked, shouldn't it be possible for a single kernel to satisfy many of these use cases?

>>There's no theoretical reason why an operating system couldn't support all of these resource-isolation use cases

>>the chroot() system call did not change the fact that the hierarchical relationship of the mounts in the filesystem was global to all processes. By contrast, mount namespaces allow different groups of processes to see different filesystem hierarchies.

Glauber's "realistic" estimation seems very close now.

***
[Linux capabilities support for user namespaces](https://lwn.net/Articles/420624/)

***
[Anatomy of a user namespaces vulnerability](https://lwn.net/Articles/543273/)
[CVE-2013-1858](https://nvd.nist.gov/vuln/detail/CVE-2013-1858)
[Code](https://lwn.net/Articles/543509/)

Fixed by 3.8.3 and 3.9, disallow the combination of CLONE_NEWUSER and CLONE_FS

The execution diagram clearly explain the procedure of the privildge exploit.
The key idea is:
* In the new userspace with the clone fs, replace your program which grants you priviledge role with
a shared lib will be linked to a set-user-ID-root program.
* Execute the set-user-ID-root program in the original namespace, and the hacked shared lib will be linked
and executed with priviledge capability in the original namespace.
The target program in the orignal namespace in turn gain the priviledge capablity.
