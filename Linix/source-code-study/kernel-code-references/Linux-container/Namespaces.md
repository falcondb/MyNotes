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

>Signals can also be sent to the PID namespace init process by processes in ancestor PID namespaces. Again, *only the signals for which the init process has established a handler can be sent, with two exceptions: SIGKILL and SIGSTOP*. When a process in an ancestor PID namespace sends these two signals to the init process, they are forcibly delivered (and can't be caught). The SIGSTOP signal stops the init process; SIGKILL terminates it. Since the init process is essential to the functioning of the PID namespace, *if the init process is terminated by SIGKILL (or it terminates for any other reason), the kernel terminates all other processes in the namespace by sending them a SIGKILL signal*.

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
>
***
[User namespaces progress](https://lwn.net/Articles/528078/)


***
[LCE: The failure of operating systems and how we can fix it](https://lwn.net/Articles/524952/)
