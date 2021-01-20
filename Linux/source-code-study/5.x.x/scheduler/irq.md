## IRQ

### softirq

#### Key data structures
##### `interrupt.h`
* `softirq`
```
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};


static struct smp_hotplug_thread softirq_threads = {
	.store			= &ksoftirqd,
	.thread_should_run	= ksoftirqd_should_run,
	.thread_fn		= run_ksoftirqd,
	.thread_comm		= "ksoftirqd/%u",
};

```
* `tasklet`
```
/* Tasklets --- multithreaded analogue of BHs.

   Main feature differing them of generic softirqs: tasklet
   is running only on one CPU simultaneously.

   Main feature differing them of BHs: different tasklets
   may be run simultaneously on different CPUs.

   Properties:
   * If tasklet_schedule() is called, then tasklet is guaranteed
     to be executed on some cpu at least once after this.
   * If the tasklet is already scheduled, but its execution is still not
     started, it will be executed only once.
   * If this tasklet is already running on another CPU (or schedule is called
     from tasklet itself), it is rescheduled for later.
   * Tasklet is strictly serialized wrt itself, but not
     wrt another tasklets. If client needs some intertask synchronization,
     he makes it with spinlocks.
 */

struct tasklet_struct
{
	struct tasklet_struct *next;
	unsigned long state;
	atomic_t count;
	void (*func)(unsigned long);		// callback
	unsigned long data;							// Parameter for the callback
};
```

#### `kernel/softirq.c`
```
__init softirq_init
 	for each cpu
		per_cpu(tasklet_vec, cpu).tail = &per_cpu(tasklet_vec, cpu).head;
		per_cpu(tasklet_hi_vec, cpu).tail = &per_cpu(tasklet_hi_vec, cpu).head;

	open_softirq(TASKLET_SOFTIRQ, tasklet_action);
	open_softirq(HI_SOFTIRQ, tasklet_hi_action);
```

```
__init spawn_ksoftirqd
	cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL, takeover_tasklets);

early_initcall(spawn_ksoftirqd)
```

```
open_softirq
  softirq_vec[nr].action = action
```

```
do_softirq
  do_softirq_own_stack
    arch dependent x86_32:
      /* build the stack frame on the softirq stack */
      isp = irqstk + sizeof(*irqstk)
      /* Push the previous esp onto the stack */
      *irqstk = current_stack_pointer
      call_on_stack(__do_softirq, isp)  // call_on_stack is the assemble code      
```

```
__do_softirq
  struct softirq_action *h
  h->action(h)
```

```
irq_enter ==> __irq_enter
irq_exit

```

```
raise_softirq ==> local_irq_save; raise_softirq_irqoff
  __raise_softirq_irqoff
    or_softirq_pending(1UL << nr)   // enable the nr softirq
```

##### Tasklet
`__tasklet_XXXX_common` is shared by `TASKLET_SOFTIRQ` and `TASKLET_HI_SOFTIRQ`
```
tasklet_schedule
	if !test_and_set_bit(TASKLET_STATE_SCHED, &t->state)
		__tasklet_schedule
			__tasklet_schedule_common(t, &tasklet_vec, TASKLET_SOFTIRQ)
				add the struct tasklet_struct to the tail of struct tasklet_head __percpu
				raise_softirq_irqoff(softirq_nr)
```

```
tasklet_action
	tasklet_action_common
		for each tasklet_struct t in the tasklet_head.head
			t->func(t->data)

		hoursekeeping of the tasklet_head
		__raise_softirq_irqoff
```

##### Workqueues
* `workqueue.h`
```
struct work_struct {
	atomic_long_t data;
	struct list_head entry;
	work_func_t func;
	struct lockdep_map lockdep_map;
};
```

TOBESTUDIED
* `kernel/workqueue.c`
```
__init workqueue_init

```

```
queue_work
```
