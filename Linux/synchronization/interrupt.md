## interrupt handling

[Linux Device Driver: Chapter 10](https://static.lwn.net/images/pdf/LDD3/ch10.pdf)

* Disable interrupt delievery

disable interrupt delivery for a specific interrupt line. Calling any of these functions may update the mask for the specified irq in the programmable interrupt controller (PIC), thus disabling or enabling the specified IRQ across all processors.
```
void disable_irq(int irq);
void disable_irq_nosync(int irq);
void enable_irq(int irq);
```

A call to `local_irq_save` disables interrupt delivery on the current processor after saving the current interrupt state into flags. `local_irq_disable` shuts off local interrupt delivery without saving the state

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

[Understanding the Linux Kernel, 3rd Edition](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/ch04s07.html)
- raise_softirq
  - local_irq_save
  - Marks the softirq as pending by setting the bit corresponding to the index nr in the softirq bit mask of the local CPU
  - if in_interrupt(), clean up and return
  - wakeup_softirqd
  - local_irq_restore

- do_softirq when local_softirq_pending()
    - if in_interrupt, return
    - local_irq_save
    - If the size of the thread_union structure is 4 KB, it switches to the soft IRQ stack
    - __do_softirq
    - local_irq_restore

- __do_softirq
    - Initializes the iteration counter to 10
    - Copies the softirq bit mask of the local CPU (raise_softirq could be raised again when processing the current pending softirqs)
    - local_bh_disable
    - Clears the softirq bitmap of the local CPU, so that new softirqs can be activated
    - local_irq_enable
    - For each bit set in the pending local variable, it executes the corresponding softirq function.
    - local_irq_disable
    - Copies the softirq bit mask of the local CPU into the pending local variable and decreases the iteration counter one more time
    - if pending is not zero, back to step 4
    - If there are more pending softirqs, it invokes wakeup_softirqd
    - Subtracts 1 from the softirq counter

- points invokes `raise_softirq`
  - `local_bh_enable`
  - `do_IRQ` finishes, and call `irq_exit`
  - With I/O APIC, when the `smp_apic_timer_interrupt` finishes
  - In SMP, after `CALL_FUNCTION_VECTOR` interprocessor interrupt
  - `ksoftirqd` is awakened.

- ksoftirqd
    ```
    for(;;) {
        set_current_state(TASK_INTERRUPTIBLE );
        schedule( );
        /* now in TASK_RUNNING state */
        while (local_softirq_pending( )) {
            preempt_disable();
            do_softirq( );
            preempt_enable();
            cond_resched( );
        }
    }
    ```
- Tasklets
    - tasklets are built on top of two softirqs named HI_SOFTIRQ and TASKLET_SOFTIRQ
    - `state` of a tasklet: TASKLET_STATE_SCHED; TASKLET_STATE_RUN
    - `tasklet_schedule` or `tasklet_hi_schedule`
      - if TASKLET_STATE_SCHED, return. the tasklet has already been scheduled
      - local_irq_save
      - Adds the tasklet descriptor at the beginning of the list pointed to by tasklet_vec[n] or tasklet_hi_vec[n], where n denotes the logical number of the local CPU.
      - raise_softirq_irqoff
      - local_irq_restore

    - `tasklet_hi_action` for HI_SOFTIRQ; `tasklet_action` for TASKLET_SOFTIRQ
        - Disables local interrupts.
        - Gets the logical number n of the local CPU.
        - Stores the address of the list pointed to by tasklet_vec[n] or tasklet_hi_vec[n] in the list local variable.
        - Puts a NULL address in tasklet_vec[n] or tasklet_hi_vec[n], thus emptying the list of scheduled tasklet descriptors.
        - Enables local interrupts.
        - For each tasklet descriptor
          - if TASKLET_STATE_RUN, execution of the tasklet is deferred until no other tasklets of the same type are running on other CPUs
          - Otherwise,  sets the flag
          - Checks whether the tasklet is disabled by looking at the count field
          - If the tasklet is enabled, it clears the TASKLET_STATE_SCHED flag and executes the tasklet function

[Understanding Linux Network Internals](https://www.amazon.com/Understanding-Linux-Network-Internals-Networking-ebook/dp/B0043EWV3S)

| Function	| Description	|
| ---				| ---					|
|in_interrupt	| returns TRUE if the CPU is currently serving a hardware or software interrupt, or preemption is disabled.|
| in_softirq  |  returns TRUE if the CPU is currently serving a software interrupt	|
| in_irq in_irq | returns TRUE if the CPU is currently serving a hardware interrupt. |
| softirq_pending | Returns TRUE if there is at least one softirq pending (i.e., scheduled for execution) for the CPU whose ID was passed as the input argument. |
|local_softirq_pending | Returns TRUE if there is at least one softirq pending for the local CPU. |
| __raise_softirq_irqoff| Sets the flag associated with the input softirq type to mark it pending. |
|raise_softirq_irqoff| This is a wrapper around __raise_softirq_irqoff that also wakes up ksoftirqd when in_interrupt( ) returns FALSE.|
| raise_softirq| |a wrapper around raise_softirq_irqoff that disables hardware interrupts before calling it and restores them to their original status. |
|__local_bh_enable, local_bh_enable | __local_bh_enable enables bottom halves (and thus softirqs/tasklets) on the local CPU, and local_bh_enable also invokes invoke_softirq if any softirq is pending and in_interrupt( ) returns FALSE.|
| spin_lock_bh | Acquire and release a spinlock, respectively. Both functions disable and then reenable bottom halves and preemption during the operation. |
| preempt_enable	| enables preemption, checks whether the counter is zero and forces a call to schedule( ) to allow any higher-priority task to run|
| preempt_enable_no_resched | simply decrements a reference counter, which allows preemption to be re-enabled when it reaches zero. |

[Software interrupts and realtime](https://lwn.net/Articles/520076/)
