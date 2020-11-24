## task_fd_user.c
Load BPF through perf and ftrace
`bpf_task_fd_query` makes `sys_bpf(BPF_TASK_FD_QUERY, att, ...)` to query the BPF fd?!

For kprobe,
Attach BPF to Perf events without using ftrace DEBUGFS

_open()_ the `/sys/bus/event_source/devices/kprobe/type`
`sys_perf_event_open` that makes the syscall `__NR_perf_event_open 241` returns the perf event fd. We can specify the kprobe name or the address of the event function address by query /proc/kallsym
```
ioctl(fd, PERF_EVENT_IOC_ENABLE
ioctl(fd, PERF_EVENT_IOC_SET_BPF
bpf_task_fd_query
```

For uprobe, _write_ the path of the executable and uprobed function offset to `/sys/kernel/debug/tracing/uprobe_events`
_open()_ the `/sys/kernel/debug/tracing/events/uprobe/${event_alias}/id` to get the fd of the _uprobe_, _read()_ the fd for `struct perf_event_attr.config`
```
sys_perf_event_open
ioctl(fd, PERF_EVENT_IOC_ENABLE
ioctl(fd, PERF_EVENT_IOC_SET_BPF
bpf_task_fd_query
```
