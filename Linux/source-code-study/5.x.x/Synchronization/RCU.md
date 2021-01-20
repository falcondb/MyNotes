## RCU

### Key data structures

### Key functions
* `linux/types.h`
```
/**
 * struct callback_head - callback structure for use with RCU and task_work
 * @next: next update requests in a list
 * @func: actual update function to call after the grace period.
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

```
__rcu_reclaim

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

```
/* Record an rcu quiescent state.  */
rcu_qs
  // local irq disabled
  if rcu_ctrlblk.donetail != rcu_ctrlblk.curtail  // more work to do
    rcu_ctrlblk.donetail = rcu_ctrlblk.curtail
    raise_softirq_irqoff(RCU_SOFTIRQ)             // enable RCU softirq
```

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

#### rcupdate.h
```
/**
 * rcu_assign_pointer() - assign to RCU-protected pointer
 * @p: pointer to assign to
 * @v: value to assign (publish)
 *
 * Assigns the specified value to the specified RCU-protected
 * pointer, ensuring that any concurrent RCU readers will see
 * any prior initialization.
 *
 * Inserts memory barriers on architectures that require them
 * (which is most of them), and also prevents the compiler from
 * reordering the code that initializes the structure after the pointer
 * assignment.  More importantly, this call documents which pointers
 * will be dereferenced by RCU read-side code.
 *
 * In some special cases, you may use RCU_INIT_POINTER() instead
 * of rcu_assign_pointer().  RCU_INIT_POINTER() is a bit faster due
 * to the fact that it does not constrain either the CPU or the compiler.
 * That said, using RCU_INIT_POINTER() when you should have used
 * rcu_assign_pointer() is a very bad thing that results in
 * impossible-to-diagnose memory corruption.  So please be careful.
 * See the RCU_INIT_POINTER() comment header for details.
 *
 * Note that rcu_assign_pointer() evaluates each of its arguments only
 * once, appearances notwithstanding.  One of the "extra" evaluations
 * is in typeof() and the other visible only to sparse (__CHECKER__),
 * neither of which actually execute the argument.  As with most cpp
 * macros, this execute-arguments-only-once property is important, so
 * please be careful when making changes to rcu_assign_pointer() and the
 * other macros that it invokes.
 */
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

#### SRCU
Sleepable Read-Copy Update mechanism for mutual exclusion, tiny version for non-preemptible single-CPU use

* `kernel/rcu/srcutiny.c`
```
__init srcu_init

```
