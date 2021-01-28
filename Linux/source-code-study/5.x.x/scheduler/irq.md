## IRQ

### machine instructions
#### x86
`SIDT` stores the content the interrupt descriptor table register (IDTR) in the destination operand.
`LGDT/LIDT` loads the values in the source operand into the global descriptor table register (GDTR) or the interrupt descriptor table register (IDTR).

### softirq

#### Key data structures
##### `interrupt.h`
```
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};

```

#### `kernel/softirq.c`
```
__init softirq_init
	for each cpu
		per_cpu(tasklet_vec, cpu).tail = &per_cpu(tasklet_vec, cpu).head
		per_cpu(tasklet_hi_vec, cpu).tail = &per_cpu(tasklet_hi_vec, cpu).head

	open_softirq(TASKLET_SOFTIRQ, tasklet_action)
	open_softirq(HI_SOFTIRQ, tasklet_hi_action)
```

```
open_softirq
  softirq_vec[nr].action = action
```

```
do_softirq
	if in_interrupt() 	// the current CPU is serving hardware of software irq
		return

	local_irq_save

	// if any pending softirq and ksoftirqd is not scheduled
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

#### ksoftirqd
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
run_ksoftirqd
	local_irq_disable

	if local_softirq_pending
		__do_softirq
		cond_resched
```


```
__init int spawn_ksoftirqd(void)
	cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL, takeover_tasklets);
```

#### Tasklet
```
tasklet_action	or tasklet_hi_action
	tasklet_action_common
		for struct tasklet_struct *t in tasklet_head->head
			t->func(t->data)

		// put the tasklet back to the tasklet_head if the tasklet is not able to run
		// call __raise_softirq_irqoff
```

#### `kernel/smpboot.c`
