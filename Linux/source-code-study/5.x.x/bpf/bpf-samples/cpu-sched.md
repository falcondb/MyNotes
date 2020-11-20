
## cpustat linux/samples/bpf/cpustat_kern.c
Attach point tracepoint/power/cpu_idle
Perf event /sys/kernel/debug/tracing/events/power/cpu_idle

Attach point tracepoint/power/cpu_frequency
Perf event /sys/kernel/debug/tracing/events/power/cpu_frequency

### pstate vs cstate
> p-states (optimization of the voltage and CPU frequency during operation) and c-states (optimization of the power consumption if a core does not have to execute any instructions)

* pstate: During the execution of code, the operating system and CPU can optimize power consumption through different p-states (performance states). Depending on the requirements, a CPU is operated at different frequencies. P0 is the highest frequency
* cstate: C-States are used to optimize or reduce power consumption in idle mode

### cpu state statics
```
cstate = ctx->state
bpf_ktime_get_ns

bpf_map_lookup_elem for pre-p/c-state
__sync_fetch_and_add
```

## cpustat userspace linux/samples/bpf/cpustat_user.c

```
sysconf(_SC_NPROCESSORS_CONF) // get configuration information at run time, _SC_NPROCESSORS_CONF: the number of processors configured
sched_getcpu                  // determine CPU on which the calling thread is running
sched_getaffinity             // a thread's CPU affinity mask
CPU_SET                       // set current CPU to the CPU mask
sched_setaffinity
```

## preempt latency
Attach points: kprobe/trace_preempt_on, kprobe/trace_preempt_off

## wakeup latency
Attach points: kprobe/try_to_wake_up, tracepoint/sched/sched_switch
```
bpf_get_current_comm
bpf_get_stackid

```
User space print kernel stack by walking through the IP addresses in the stack.
The kernel IP addresses to symbols by calling _ksym_search_, which binary search the symbols read from _/proc/kallsyms_


## References
[tracing events power](https://www.kernel.org/doc/Documentation/trace/events-power.txt)
