## RCU

### Key data structures

* `linux/types.h`
```
/**
 *
 * The struct is aligned to size of pointer. On most architectures it happens
 * naturally due ABI requirements, but some architectures (like CRIS) have
 * weird ABI and we need to ask it explicitly.
 *
 * The alignment is required to guarantee that bit 0 of @next will be
 * clear under normal conditions -- as long as we use call_rcu() or
 * call_srcu() to queue the callback.
 *
 * This guarantee is important for few reasons:
 *  - future call_rcu_lazy() will make use of lower bits in the pointer;
 *  - the structure shares storage space in struct page with @compound_head,
 *    which encode PageTail() in bit 0. The guarantee is needed to avoid
 *    false-positive PageTail().
 */
struct callback_head {
	struct callback_head *next;
	void (*func)(struct callback_head *head);
} __attribute__((aligned(sizeof(void *))));
#define rcu_head callback_head
```

* `rcupdate_wait.h`
```
struct rcu_synchronize {
	struct rcu_head head;
	struct completion completion;  // for the GP synchronization
};
```

* `tree.h`
```

struct rcu_data {

}
```


#### RCU interfaces
* `rcu_read_lock` in rcupdate.h
```
rcu_read_lock
  __rcu_read_lock
    preempt_disable
```
```
rcu_read_unlock
  __rcu_read_unlock
    preempt_enable  
```
* `tree.c`
```
synchronize_rcu(void)
	if rcu_blocking_is_gp
      might_sleep
      preempt_disable
      ret = num_online_cpus() <= 1
      preempt_enable
		return
	if rcu_gp_is_expedited
		synchronize_rcu_expedited

  /* wait for GP has been elapsed
  else
		wait_rcu_gp(call_rcu)   // rcupdate_wait.h
      _wait_rcu_gp          // rcupdate_wait.h
        call_rcu_func_t __crcu_array[] = { call_rcu }
        struct rcu_synchronize __rs_array[ARRAY_SIZE(__crcu_array)]

        __wait_rcu_gp(checktiny, ARRAY_SIZE(__crcu_array), __crcu_array, __rs_array)
          for i in ARRAY_SIZE(__crcu_array)
            init_completion(&rs_array[i].completion)
            (crcu_array[i])(&rs_array[i].head, wakeme_after_rcu)    // crcu_array[i] should be call_rcu

            wait_for_completion(&rs_array[i].completion)  
```

```
wakeme_after_rcu(struct rcu_head *head)
	rcu = container_of(head, struct rcu_synchronize, head)
	complete(&rcu->completion)
```

```
call_rcu  ==> __call_rcu(head, func, 0)

```

```
__call_rcu

```


### Tree RCU
* `kernel/rcu/tree.c`
```
__init rcu_init
  for_each_online_cpu
    rcutree_prepare_cpu
    rcu_cpu_starting
    rcutree_online_cpu

  alloc_workqueue("rcu_gp", WQ_MEM_RECLAIM, 0)
  alloc_workqueue("rcu_par_gp", WQ_MEM_RECLAIM, 0)
  srcu_init  
```

### RCU update.c
#### rcupdate.h
```
#define rcu_assign_pointer(p, v)					      \
({									      \
	uintptr_t _r_a_p__v = (uintptr_t)(v);				      \
									      \
	if (__builtin_constant_p(v) && (_r_a_p__v) == (uintptr_t)NULL)	      \
		WRITE_ONCE((p), (typeof(p))(_r_a_p__v));		      \
	else								      \
		smp_store_release(&p, RCU_INITIALIZER((typeof(p))_r_a_p__v)); \
	_r_a_p__v;							      \
})

```

#### update.c
```
__init rcu_spawn_tasks_kthread
	t = kthread_run(rcu_tasks_kthread, NULL, "rcu_tasks_kthread");
	smp_mb(); /* Ensure others see full kthread. */
	WRITE_ONCE(rcu_tasks_kthread_ptr, t);

core_initcall(rcu_spawn_tasks_kthread);
```
Create a *kthread* with function `rcu_tasks_kthread`

```
rcu_tasks_kthread

```



### Tiny RCU, uniprocessor-only
[rcu: Add a TINY_PREEMPT_RCU](https://lwn.net/Articles/396767/)
* `kernel/rcu/tiny.c`
```
struct rcu_ctrlblk {
	struct rcu_head *rcucblist;	/* List of pending callbacks (CBs). */
	struct rcu_head **donetail;	/* ->next pointer of last "done" CB. */
	struct rcu_head **curtail;	/* ->next pointer of last CB. */
};

/* Definition for rcupdate control block. */
static struct rcu_ctrlblk rcu_ctrlblk = {
	.donetail	= &rcu_ctrlblk.rcucblist,
	.curtail	= &rcu_ctrlblk.rcucblist,
};
```
#### Initialization
* `kernel/rcu/tiny.c`
register a *softirq* with a function `rcu_process_callbacks`
```
__init rcu_init
  open_softirq(RCU_SOFTIRQ, rcu_process_callbacks)
  srcu_init
```

```
rcu_process_callbacks
  list = rcu_ctrlblk.rcucblist;
  rcu_ctrlblk.rcucblist = *rcu_ctrlblk.donetail
  *rcu_ctrlblk.donetail = NULL
  if rcu_ctrlblk.curtail == rcu_ctrlblk.donetail
    rcu_ctrlblk.curtail = &rcu_ctrlblk.rcucblist
  rcu_ctrlblk.donetail = &rcu_ctrlblk.rcucblist

  while list
		local_bh_disable
		__rcu_reclaim("", list)
		local_bh_enable
```
Split the `rcucblist` from `donetail` and reset `rcucblist` to where `donetail` pointing is.
Call `__rcu_reclaim` on every `callback_head` in the splited list before `donetail`.
```
/*
 * Reclaim the specified callback, either by invoking it (non-lazy case)
 * or freeing it directly (lazy case).  Return true if lazy, false otherwise.
 */
__rcu_reclaim
	if head->func < 4096   // __is_kfree_rcu_offset in rcuupdate.h
		kfree((void *)head - offset);
  else
		head->func(head)
```

```
call_rcu(struct rcu_head *head, rcu_callback_t func)
  head->func = func
  // local irq disabled
  *rcu_ctrlblk.curtail = head;
  rcu_ctrlblk.curtail = &head->next;

  /* force scheduling for rcu_qs() if current thread is idle*/
  resched_cpu(0)
```
Populate the `struct rcu_head` pointed by `head`, add it to the current tail of the callback list `rcucblist`.  `rcu_ctrlblk.curtail` is the pointer pointing to the address of the next of current tail list.

```
/* Record an rcu quiescent state.  */
rcu_qs
  // local irq disabled
  if rcu_ctrlblk.donetail != rcu_ctrlblk.curtail  // more work to do
    rcu_ctrlblk.donetail = rcu_ctrlblk.curtail
    raise_softirq_irqoff(RCU_SOFTIRQ)             // enable RCU softirq
```
`donetail` jumps to `curtail` and raise softirq `RCU_SOFTIRQ`.

```
rcu_barrier
  wait_rcu_gp

```



#### SRCU
Sleepable Read-Copy Update mechanism for mutual exclusion, tiny version for non-preemptible single-CPU use

* `kernel/rcu/srcutiny.c`
```
__init srcu_init

```
