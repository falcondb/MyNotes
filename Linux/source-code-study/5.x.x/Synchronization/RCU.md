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
/*
 * Definition for node within the RCU grace-period-detection hierarchy.
 */
struct rcu_node {
  raw_spinlock_t __private lock;	/* Root rcu_node's lock protects */
  unsigned long gp_seq;	/* Track rsp->rcu_gp_seq. */
  unsigned long gp_seq_needed; /* Track furthest future GP request. */
  unsigned long completedqs; /* All QSes done for this node. */
  ...
  struct rcu_node *parent;
  ...
}  
```    

```
/* Per-CPU data for read-copy update. */
struct rcu_data {
  /* 1) quiescent-state and grace-period handling : */
  unsigned long	gp_seq;		/* Track rsp->rcu_gp_seq counter. */
  unsigned long	gp_seq_needed;	/* Track furthest future GP request. */
  ...
  struct rcu_node *mynode;	/* This CPU's leaf of hierarchy */
  ...
  unsigned long	ticks_this_gp;	/* The number of scheduling-clock ticks this CPU has handled during and after the last grace period it is aware of. */

  /* 2) batch handling */
  struct rcu_segcblist cblist;

  /* 3) dynticks interface. */
  ...

  /* 4) rcu_barrier(), OOM callbacks, and expediting. */
  struct rcu_head barrier_head;
  int exp_dynticks_snap;		/* Double-check need for IPI. */

  /* 5) Callback offloading. */
  ...

  /* 6) Diagnostic data, including RCU CPU stall warnings. */
  ...

  int cpu;
}
```

```
/*
 * RCU global state, including node hierarchy.  This hierarchy is
 * represented in "heap" form in a dense array.  The root (first level)
 * of the hierarchy is in ->node[0] (referenced by ->level[0]), the second
 * level in ->node[1] through ->node[m] (->node[1] referenced by ->level[1]),
 * and the third level in ->node[m+1] and following (->node[m+1] referenced
 * by ->level[2]).  The number of levels is determined by the number of
 * CPUs and by CONFIG_RCU_FANOUT.  Small systems will have a "hierarchy"
 * consisting of a single rcu_node.
 */
struct rcu_state {
	struct rcu_node node[NUM_RCU_NODES];	/* Hierarchy. */
	struct rcu_node *level[RCU_NUM_LVLS + 1];
						/* Hierarchy levels (+1 to */
						/*  shut bogus gcc warning) */
	int ncpus;				/* # CPUs seen so far. */

	/* The following fields are guarded by the root rcu_node's lock. */

	u8	boost ____cacheline_internodealigned_in_smp;
						/* Subject to priority boost. */
	unsigned long gp_seq;			/* Grace-period sequence #. */
	struct task_struct *gp_kthread;		/* Task for grace periods. */
	struct swait_queue_head gp_wq;		/* Where GP task waits. */
	short gp_flags;				/* Commands for GP task. */
	short gp_state;				/* GP kthread sleep state. */

	/* End of fields guarded by root rcu_node's lock. */

	struct mutex barrier_mutex;		/* Guards barrier fields. */
	atomic_t barrier_cpu_count;		/* # CPUs waiting on. */
	struct completion barrier_completion;	/* Wake at barrier end. */
	unsigned long barrier_sequence;		/* ++ at start and end of */

  ...
}

```

* `rcu_segcblist.h`
```
/* Complicated segmented callback lists.  ;-) */

/*
 * Index values for segments in rcu_segcblist structure.
 *
 * The segments are as follows:
 *
 * [head, *tails[RCU_DONE_TAIL]):
 *	Callbacks whose grace period has elapsed, and thus can be invoked.
 * [*tails[RCU_DONE_TAIL], *tails[RCU_WAIT_TAIL]):
 *	Callbacks waiting for the current GP from the current CPU's viewpoint.
 * [*tails[RCU_WAIT_TAIL], *tails[RCU_NEXT_READY_TAIL]):
 *	Callbacks that arrived before the next GP started, again from
 *	the current CPU's viewpoint.  These can be handled by the next GP.
 * [*tails[RCU_NEXT_READY_TAIL], *tails[RCU_NEXT_TAIL]):
 *	Callbacks that might have arrived after the next GP started.
 *	There is some uncertainty as to when a given GP starts and
 *	ends, but a CPU knows the exact times if it is the one starting
 *	or ending the GP.  Other CPUs know that the previous GP ends
 *	before the next one starts.
 *
 * Note that RCU_WAIT_TAIL cannot be empty unless RCU_NEXT_READY_TAIL is also
 * empty.
 *
 * The ->gp_seq[] array contains the grace-period number at which the
 * corresponding segment of callbacks will be ready to invoke.  A given
 * element of this array is meaningful only when the corresponding segment
 * is non-empty, and it is never valid for RCU_DONE_TAIL (whose callbacks
 * are already ready to invoke) or for RCU_NEXT_TAIL (whose callbacks have
 * not yet been assigned a grace-period number).
 */
#define RCU_DONE_TAIL		0	/* Also RCU_WAIT head. */
#define RCU_WAIT_TAIL		1	/* Also RCU_NEXT_READY head. */
#define RCU_NEXT_READY_TAIL	2	/* Also RCU_NEXT head. */
#define RCU_NEXT_TAIL		3
#define RCU_CBLIST_NSEGS	4

struct rcu_segcblist {
	struct rcu_head *head;
	struct rcu_head **tails[RCU_CBLIST_NSEGS];
	unsigned long gp_seq[RCU_CBLIST_NSEGS];
	long len;
	long len_lazy;
};
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


### Tree RCU
Refer the _RCU data structure_ section in note _RCU.md_.
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
  rcu_segcblist_enqueue

  __call_rcu_core
  note_gp_changes


```

```
note_gp_changes
  needwake = __note_gp_changes
  if needwake
		rcu_gp_kthread_wake

```

```
/*
 * Update CPU-local rcu_data state to record the beginnings and ends of
 * grace periods.  The caller must hold the ->lock of the leaf rcu_node
 * structure corresponding to the current CPU, and must have irqs disabled.
 */
__note_gp_changes
  /* Handle the ends of any preceding grace periods first. */
    if rcu_seq_completed_gp(rdp->gp_seq, rnp->gp_seq) ==> ULONG_CMP_LT(old, new & ~RCU_SEQ_STATE_MASK)  ==> ULONG_MAX / 2 < (a) - (b)
    rcu_advance_cbs
      rcu_segcblist_advance(&rdp->cblist, rnp->gp_seq)  // see section rcu_segcblist.c

      rcu_accelerate_cbs
  else
    rcu_accelerate_cbs

  /* Now handle the beginnings of any new-to-this-CPU grace periods. */  
  if (rcu_seq_new_gp(rdp->gp_seq, rnp->gp_seq)
    need_gp = !!(rnp->qsmask & rdp->grpmask)
    rdp->cpu_no_qs.b.norm = need_gp
    rdp->core_needs_qs = need_gp
    zero_cpu_stall_ticks(rdp)

  rdp->gp_seq = rnp->gp_seq

  rcu_gpnum_ovf  
```

```
/*
 * Check to see if this CPU is in a non-context-switch quiescent state
 * (user mode or idle loop for rcu, non-softirq execution for rcu_bh).
 * Also schedule RCU core processing.
 *
 * This function must be called from hardirq context.  It is normally
 * invoked from the scheduling-clock interrupt.
 */
rcu_check_callbacks

```

```
/*
 * Check to see if there is a new grace period of which this CPU
 * is not yet aware, and if so, set up local rcu_data state for it.
 * Otherwise, see if this CPU has just passed through its first
 * quiescent state for this grace period, and record that fact if so.
 */
rcu_check_quiescent_state

```

### `rcu_segcblist.c`
```
rcu_segcblist_advance


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
