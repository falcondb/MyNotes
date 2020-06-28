struct perf_event in include/linux/perf_event.h
==================
```

```


perf_tp_event() in /kernel/events/core
==================
```
perf_tp_event() ==>
hlist_for_each_entry_rcu(event, head, hlist_entry) {
  if (perf_tp_event_match(event, &data, regs))
    perf_swevent_event(event, count, &data, regs);
}

```


```
perf_kprobe_event_init {
  .event_init	= perf_kprobe_event_init,
}

perf_uprobe_event_init {
  .event_init	= perf_uprobe_event_init,
}
perf_kprobe_event_init() ==>  perf_kprobe_init()
perf_uprobe_event_init() ==>  perf_uprobe_init()

perf_tp_register()  ==>  perf_pmu_register

bpf_overflow_handler ==>  BPF_PROG_RUN ;  event->orig_overflow_handler  ???

perf_event_set_bpf_handler  ==> 	prog = bpf_prog_get_type(prog_fd, BPF_PROG_TYPE_PERF_EVENT);  event->prog = prog



```

perf_event_set_bpf_prog  in /kernel/events/core
==================
```
perf_event_set_bpf_prog  ==>  perf_event_attach_bpf_prog ==> event->prog = prog
```


perf_event_open syscall
===================
```
list_add_tail(&event->owner_entry, &current->perf_event_list);
```
