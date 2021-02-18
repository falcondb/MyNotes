## interrupt handling

[Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/)
ksoftirqd system is initialized in `kernel/softirq.c`
```
DEFINE_PER_CPU(struct task_struct *, ksoftirqd);

static struct smp_hotplug_thread softirq_threads = {
	.store			= &ksoftirqd,
	.thread_should_run	= ksoftirqd_should_run,
	.thread_fn		= run_ksoftirqd,
	.thread_comm		= "ksoftirqd/%u",
};

```

```
run_ksoftirqd
  if local_softirq_pending()
    __do_softirq()
      struct softirq_action h->action(h)
    local_irq_enable()
    cond_resched()
      // TOBESTUDIED
```

```
__init int spawn_ksoftirqd(void)
	cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL, takeover_tasklets);
```

[Linux Device Driver: Chapter 10](https://static.lwn.net/images/pdf/LDD3/ch10.pdf)

* Disable interrupt delievery

disable interrupt delivery for a specific interrupt line. Calling any of these functions may update the maskfor the specified irq in the programmable
interrupt controller (PIC), thus disabling or enabling the specified IRQ across all processors.
```
void disable_irq(int irq);
void disable_irq_nosync(int irq);
void enable_irq(int irq);
```

A call to `local_irq_save` disables interrupt delivery on the current processor after saving
the current interrupt state into flags. `local_irq_disable` shuts off local interrupt delivery without saving the state

```
void local_irq_save(unsigned long flags);
void local_irq_disable(void);
```
* Top and Bottom Halves
> top half is the routine that actually responds to the interrupt—the one you register with request_irq. The bottom half is a routine that is scheduled by the top half to be executed later, at a safer time. Top half is the routine that actually responds to the interrupt—the one you register with request_irq. The bottom half is a routine that is scheduled by the top half to be executed later, at a safer time.

> Tasklets are often the preferred mechanism for bottom-half processing; they are very fast, but all tasklet code must be atomic. The alternative to tasklets is workqueues, which may have a higher latency but that are allowed to sleep.

* Tasklets
> tasklets are a special function that may be scheduled to run, in software interrupt context, at a system-determined safe time.

> They may be scheduled to run multiple times, but tasklet scheduling is not cumulative; the tasklet runs only once, even if it is requested repeatedly before it is launched. No tasklet ever runs in parallel with itself, since they run only once, but tasklets can run in parallel with other tasklets on SMP systems. They may be scheduled to run multiple times, but tasklet scheduling is not cumulative; the tasklet runs only once, even if it is requested repeatedly before it is launched. No tasklet ever runs in parallel with itself, since they run only once, but tasklets can run in parallel with other tasklets on SMP systems.

* Workqueues
> workqueues invoke a function at some future time in the context of a special worker process. Since the workqueue function runs in process context, it can sleep if need be.


[Introduction to deferred interrupts](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-9.html)
All new schemes of implementation of the bottom half handlers are built on the performance of the processor specific kernel thread that called `ksoftirqd`.
Softirqs are determined statically at compile-time of the Linux kernel and the `open_softirq` function takes care of softirq initialization.

`tasklets` are `softirqs` that can be allocated and initialized at runtime, `tasklets` that have the same type cannot be run on multiple processors at a time.

`Workqueue` functions run in the context of a kernel process, but tasklet functions run in the software interrupt context. This means that workqueue functions must not be atomic as tasklet functions.
The work queue subsystem is an interface for creating kernel threads to handle work that is queued from elsewhere. All of these kernel threads are called -- worker threads.
