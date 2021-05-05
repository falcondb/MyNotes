## RCU
Better to read "Verification of the Tree-Based Hierarchical Read-Copy Update in the Linux Kernel" first

### High level idea of how RCU tree works from my own study on v5.12.x
* How QS is collected from the CPUs:
	- The QS state is collected from the current CPU by calling `rcu_qs()`, which basically sets `rcu_data.cpu_no_qs.b.norm` to false.
	- In the Softirq RCU mode (`module_param(use_softirq, bool, 0444)`), after Softirq handler `__do_softirq` finishing the pending Softirq events, it is a QS state, and Softirq handler calls `rcu_softirq_qs ==> rcu_qs` to report this CPU entered QS state in `rcu_data`.
	- The RCU Softirq `RCU_SOFTIRQ` handler `rcu_core_si and rcu_core` works in the following steps:
		- `note_gp_changes` updates itself for any GP changes including updating the callbacks, reset `cpu_no_qs.b.norm` if parent doesn't need it, and update GP in `rcu_data`
		- `rcu_report_qs_rdp` propagates the RCU tree nodes' state changes.
		- finally, `rcu_do_batch` calls the callbacks registered before this GP

* How the GPs are managed:
	- There are kernel threads designed for the GP management. They are created during the RCU initialization `__init rcu_spawn_gp_kthread`, the kernel thread for plain GP management `rcu_gp_kthread` is created here (other RCU threads with special missions `rcu_nocb_cb_kthread, rcu_boost_kthread` are also created here, but not current main study focus).
	- `rcu_boost_kthread` manages the GPs, the high-level procedure of GP management:
		- loops with `rcu_gp_init` util GP initialization is completed
		- loops in `rcu_gp_fqs_loop ==> rcu_gp_fqs`, loop doing repeated quiescent-state forcing until the grace period ends. It keeps asking the leaf nodes to `rcu_report_qs_rnp`
		- `rcu_gp_cleanup` generates the next GP seq, updates the `gp_seq` with the new GP seq. Call `__note_gp_changes` for this CPU where `rcu_gp_kthread` is running. Other clean-up work.

### Key data structures

* `linux/types.h`
```
struct callback_head {
	struct callback_head *next;
	void (*func)(struct callback_head *head);
} __attribute__((aligned(sizeof(void *))));
#define rcu_head callback_head

typedef void (*rcu_callback_t)(struct rcu_head *head);
typedef void (*call_rcu_func_t)(struct rcu_head *head, rcu_callback_t func);

```

* `rcupdate_wait.h`
```
struct rcu_synchronize {
	struct rcu_head head;
	struct completion completion;  // for the GP synchronization
};
```

* `kernel/rcu/tree.h`
```
/* Values for rcu_state structure's gp_state field. */
#define RCU_GP_IDLE	 0	/* Initial state and no GP in progress. */
#define RCU_GP_WAIT_GPS  1	/* Wait for grace-period start. */
#define RCU_GP_DONE_GPS  2	/* Wait done for grace-period start. */
#define RCU_GP_ONOFF     3	/* Grace-period initialization hotplug. */
#define RCU_GP_INIT      4	/* Grace-period initialization. */
#define RCU_GP_WAIT_FQS  5	/* Wait for force-quiescent-state time. */
#define RCU_GP_DOING_FQS 6	/* Wait done for force-quiescent-state time. */
#define RCU_GP_CLEANUP   7	/* Grace-period cleanup started. */
#define RCU_GP_CLEANED   8	/* Grace-period cleanup complete. */
```

```
/*
 * Definition for node within the RCU grace-period-detection hierarchy.
 */
struct rcu_node {
  raw_spinlock_t __private lock;	/* Root rcu_node's lock protects */
  unsigned long gp_seq;	/* Track rsp->rcu_gp_seq. */
  unsigned long gp_seq_needed; /* Track furthest future GP request. */
  unsigned long completedqs; /* All QSes done for this node. */

  unsigned long qsmask;
  unsigned long qsmaskinit;
  unsigned long qsmaskinitnext;


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
  if (use_softirq)
    open_softirq(RCU_SOFTIRQ, rcu_core_si)

  for_each_online_cpu
    rcutree_prepare_cpu     // rdp initialization
    rcu_cpu_starting        // setup the masks, for new incoming CPU call rcu_report_qs_rnp if waiting for its qs report
    rcutree_online_cpu      // final step for online CPU, kthread affinity

	/* Create workqueue for expedited GPs and for Tree SRCU. */
	rcu_gp_wq = alloc_workqueue("rcu_gp", WQ_MEM_RECLAIM, 0)
	rcu_par_gp_wq = alloc_workqueue("rcu_par_gp", WQ_MEM_RECLAIM, 0)

  srcu_init  
```

```
synchronize_rcu(void)
	if rcu_gp_is_expedited
		synchronize_rcu_expedited

  /* wait for GP has been elapsed
  else
		wait_rcu_gp(call_rcu)   // rcupdate_wait.h see call_rcu implementation!
      _wait_rcu_gp          // rcupdate_wait.h
        __wait_rcu_gp
          for i in ARRAY_SIZE(__crcu_array)
            // if the passed in callback is already registered, skip the callback registration
            (crcu_array[i])(&rs_array[i].head, wakeme_after_rcu)    // crcu_array[i] should be call_rcu

            wait_for_completion(&rs_array[i].completion)      // wait here for complete(&rcu->completion)
```

```
wakeme_after_rcu(struct rcu_head *head)
	rcu = container_of(head, struct rcu_synchronize, head)
	complete(&rcu->completion)
```

```
call_rcu ==> __call_rcu  // not lazy callback
  rcu_segcblist_enqueue

  __call_rcu_core
		if !rcu_is_watching
				invoke_rcu_core()
					if (use_softirq)
						raise_softirq(RCU_SOFTIRQ);		// see rcu_core_si and rcu_core, enabling RCU_SOFTIRQ for ksoftirqd
					else
						invoke_rcu_core_kthread()

```

```
/*
 * Update CPU-local rcu_data state to record the beginnings and ends of
 * grace periods.  The caller must hold the ->lock of the leaf rcu_node
 * structure corresponding to the current CPU, and must have irqs disabled.
 */
note_gp_changes ==> __note_gp_changes
	if (rdp->gp_seq == rnp->gp_seq)
		return false
  /* Handle the ends of any preceding grace periods first. */
    if rcu_seq_completed_gp(rdp->gp_seq, rnp->gp_seq) ==> ULONG_CMP_LT(old, new & ~RCU_SEQ_STATE_MASK)  ==> ULONG_MAX / 2 < (a) - (b)
    rcu_advance_cbs
      rcu_segcblist_advance(&rdp->cblist, rnp->gp_seq)  // see section rcu_segcblist.c

      rcu_accelerate_cbs
  else
    rcu_accelerate_cbs

  /* Now handle the beginnings of any new-to-this-CPU grace periods. */  
  if (rcu_seq_new_gp(rdp->gp_seq, rnp->gp_seq)	// the last two bits in gp_seq are not set
    need_gp = !!(rnp->qsmask & rdp->grpmask)
    rdp->cpu_no_qs.b.norm = need_gp		// the parent rnp doesn't need this rdp to report qs
    rdp->core_needs_qs = need_gp
    zero_cpu_stall_ticks(rdp)

  rdp->gp_seq = rnp->gp_seq

```


### `rcu_gp_kthread` in `tree.c`
#### RCU kthread init

```
/* the kthreads that handle each RCU flavor's grace periods. */
__init rcu_spawn_gp_kthread
  t= kthread_create(rcu_gp_kthread, ...)
  rcu_state.gp_kthread = t
  wake_up_process(t)
  rcu_spawn_nocb_kthreads()   // for each cpu
  rcu_spawn_boost_kthreads()  // for each cpu

early_initcall(rcu_spawn_gp_kthread);
```

#### RCU kthread main loop
```
rcu_gp_kthread
  rcu_bind_gp_kthread   // bind to hoursekeeping CPU

  for (;;)

    for (;;)
      rcu_state.gp_state = RCU_GP_WAIT_GPS
      rcu_state.gp_state = RCU_GP_DONE_GPS

      if rcu_gp_init()  // Init is done here!
        break   // jump to the end of the for the inner loop and nothing to be done right now

        cond_resched()  // see section RCU, cond_resched(), and performance regressions in RCU.md
      // end of inner loop


    /* Handle quiescent-state forcing. */  
    rcu_gp_fqs_loop

    /* Handle grace-period end. */
		rcu_state.gp_state = RCU_GP_CLEANUP;
		rcu_gp_cleanup
		rcu_state.gp_state = RCU_GP_CLEANED;

  // end of outer loop      
```

```
/*
 * Initialize a new grace period.  Return false if no grace period required.
 */
rcu_gp_init

  WRITE_ONCE(rcu_state.gp_flags, 0)  /* Clear all flags: New GP. */

  /* Advance to a new grace period and initialize state. */
  rcu_seq_start   ==>  WRITE_ONCE(&rcu_state.gp_seq + 1); smp_mb();

  rcu_state.gp_state = RCU_GP_INIT

	rcu_for_each_node_breadth_first(rnp)

    rnp->qsmask = rnp->qsmaskinit
    WRITE_ONCE(rnp->gp_seq, rcu_state.gp_seq) // update the rcu_node qp_seq

    if (rnp == rdp->mynode)
			 __note_gp_changes(rnp, rdp)   // report to its parent leaf rcu_node

		if ((mask || rnp->wait_blkd_tasks) && rcu_is_leaf_node(rnp))			 
				rcu_report_qs_rnp	 


```

```
/*
 * Loop doing repeated quiescent-state forcing until the grace period ends.
 */
rcu_gp_fqs_loop   // called by rcu_gp_kthread after handling starting a GP
	rnp = rcu_get_root()

  for (;;)
    /* If time for quiescent-state forcing, do it. */
		if (!READ_ONCE(rnp->qsmask) && !rcu_preempt_blocked_readers_cgp(rnp))
			break

    rcu_gp_fqs
        force_qs_rnp(dyntick_save_progress_counter) // for the first fqs
        or
        force_qs_rnp(rcu_implicit_dynticks_qs)    /* Handle dyntick-idle and offline CPUs. */

```

```
force_qs_rnp
  rcu_for_each_leaf_node(rnp)

      for_each_leaf_node_possible_cpu
          bit = leaf_node_cpu_bit(rnp, cpu)
          if rnp->qsmask & bit  != 0
              if f(rdp)			// f is dyntick_save_progress_counter  or   rcu_implicit_dynticks_qs
                  mask |= bit
      // end of for_each_leaf_node_possible_cpu


      if mask != 0
          rcu_report_qs_rnp(mask, rnp, rnp->gp_seq, flags)
```

```
/*
 * Clean up after the old grace period.
 */
rcu_gp_cleanup

  /* Propagate new ->gp_seq value to rcu_node structures so that
  * other CPUs don't have to wait until the start of the next grace
  * period to process their callbacks.
  */
  rcu_seq_end
  rcu_for_each_node_breadth_first
    WRITE_ONCE(rnp->gp_seq, new_gp_seq)
    if rnp == rdp->mynode
			needgp = __note_gp_changes(rnp, rdp) || needgp

    needgp = rcu_future_gp_cleanup(rnp) || needgp;

    rcu_gp_slow  
```

```
rcu_tasks_qs(current)		// for new RCU task implementation?
	WRITE_ONCE((current)->rcu_tasks_holdout, false)
```



### `rcu_segcblist.c`
```
rcu_segcblist_advance
  /*
  * Find all callbacks whose ->gp_seq numbers indicate that they
  * are ready to invoke, and put them into the RCU_DONE_TAIL segment.
  */
  for i = RCU_WAIT_TAIL; i < RCU_NEXT_TAIL; i++
    if ULONG_CMP_LT(seq, rsclp->gp_seq[i])
      break;
    WRITE_ONCE(rsclp->tails[RCU_DONE_TAIL], rsclp->tails[i]);

  /* Clean up tail pointers that might have been misordered above. */
	for (j = RCU_WAIT_TAIL; j < i; j++)
		WRITE_ONCE(rsclp->tails[j], rsclp->tails[RCU_DONE_TAIL]);  

	for j = RCU_WAIT_TAIL; i < RCU_NEXT_TAIL; i++, j++
		if rsclp->tails[j] == rsclp->tails[RCU_NEXT_TAIL]
			break;  /* No more callbacks. */
		WRITE_ONCE(rsclp->tails[j], rsclp->tails[i])
		rsclp->gp_seq[j] = rsclp->gp_seq[i]

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
rcu_core_si is the handler for RCU Softirq

rcu_core_si == rcu_core

  rcu_check_quiescent_state
    note_gp_changes

		if (rdp->cpu_no_qs.b.norm)
			return

    rcu_report_qs_rdp
				if (rdp->cpu_no_qs.b.norm || rdp->gp_seq != rnp->gp_seq || rdp->gpwrap)

						/* cpu_no_qs.b.norm == false means at least 1 qs passed see rcu_qs
						* rdp->gp_seq != rnp->gp_seq?! cpu offline?! */
						rdp->cpu_no_qs.b.norm = true;	/* need qs for new gp. */
						return

		        rcu_report_qs_rnp(mask, rnp, rnp->gp_seq, flags)    //see rcu_report_qs_rnp

  rcu_check_gp_start_stall

  rcu_do_batch
    rcu_segcblist_extract_done_cbs

    for rhp = rcu_cblist_dequeue
      __rcu_reclaim
        head->func( head )
```


```
/* called in __do_softirq() when the ksoftirqd is on current CPU after processing the pending softirqs
rcu_softirq_qs
  rcu_qs
    __this_cpu_write(rcu_data.cpu_no_qs.b.norm, false)
```


```
rcu_start_this_gp
  for rnp = rnp->parent // traverse from the given rcu_node to the root
    rnp->gp_seq_needed = gp_seq_req   // update the furthest future GP request

  WRITE_ONCE(rcu_state.gp_flags, rcu_state.gp_flags | RCU_GP_FLAG_INIT)

```


```
rcu_report_qs_rnp
  /* Walk up the rcu_node hierarchy. */
  for (;;)
    /* Our bit has already been cleared, or the relevant grace period is already over, so done, return. */
    rnp->qsmask &= ~mask;   // clear the bit from the reported child rnp
    if rnp->qsmask != 0   
      return  // Other bits still set at this level, so done

    rnp->completedqs = rnp->gp_seq;
    mask = rnp->grpmask;    // set mask to this rnp grpmask for the next for loop
    rnp = rnp->parent

		rcu_report_qs_rsp
			rcu_gp_kthread_wake
```



```
/* Task-based RCU implementations 2020 */
__init rcu_spawn_tasks_kthread
	t = kthread_run(rcu_tasks_kthread, NULL, "rcu_tasks_kthread");
	smp_mb(); /* Ensure others see full kthread. */
	WRITE_ONCE(rcu_tasks_kthread_ptr, t);

core_initcall(rcu_spawn_tasks_kthread);
```

```
rcu_tasks_kthread

```

The RCU management subsytem is triggered by *CPU tick event*
```
tick_handle_periodic ==> tick_periodic and tick_sched_handle
		update_process_times
				rcu_sched_clock_irq
					if rcu_pending(user)
							invoke_rcu_core()
									if (use_softirq)
											raise_softirq(RCU_SOFTIRQ)
									else
											invoke_rcu_core_kthread()

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
