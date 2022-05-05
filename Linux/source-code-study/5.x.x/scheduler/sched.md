## scheduler
### linux/sched.h
```
struct task_struct {
	struct thread_info		thread_info;
	volatile long			state;   	/* -1 unrunnable, 0 runnable, >0 stopped: */
	void				*stack;
	refcount_t			usage;
	unsigned int			flags;
	unsigned int			ptrace;

  struct llist_node		wake_entry;
	int				on_cpu;
	unsigned int			cpu;
	unsigned int			wakee_flips;
	unsigned long			wakee_flip_decay_ts;
	struct task_struct		*last_wakee;
	int				recent_used_cpu;
	int				wake_cpu;
	int				on_rq;
	int				prio;
	int				static_prio;
	int				normal_prio;
	unsigned int			rt_priority;

	const struct sched_class	*sched_class;
	struct sched_entity		se;
	struct sched_rt_entity		rt;
	struct task_group		*sched_task_group;
	struct sched_dl_entity		dl;
	unsigned int			policy;
	int				nr_cpus_allowed;
	const cpumask_t			*cpus_ptr;
	cpumask_t			cpus_mask;
	int				rcu_read_lock_nesting;
	union rcu_special		rcu_read_unlock_special;
	struct list_head		rcu_node_entry;
	struct rcu_node			*rcu_blocked_node;
	unsigned long			rcu_tasks_nvcsw;
	u8				rcu_tasks_holdout;
	u8				rcu_tasks_idx;
	int				rcu_tasks_idle_cpu;
	struct list_head		rcu_tasks_holdout_list;
	struct sched_info		sched_info;
	struct list_head		tasks;
	struct plist_node		pushable_tasks;
	struct rb_node			pushable_dl_tasks;

	struct mm_struct		*mm;     /* user space address mapping */
	struct mm_struct		*active_mm;  /* for lazy user or kernel thread */

	struct vmacache			vmacache;
	int				exit_state;
	int				exit_code;
	int				exit_signal;
	int				pdeath_signal;  	/* The signal sent when the parent dies: */

	/* Bit to tell LSMs we're in execve(): */
	unsigned			in_execve:1;
	unsigned			in_iowait:1;
	/* disallow userland-initiated cgroup migration */
	unsigned			no_cgroup_migration:1;
	/* task is frozen/stopped (used by the cgroup freezer) */
	unsigned			frozen:1;
	/* to be used once the psi infrastructure lands upstream. */
	unsigned			use_memdelay:1;

	unsigned long			atomic_flags; /* Flags requiring atomic access. */
	struct restart_block		restart_block;
	pid_t				pid;
	pid_t				tgid;

	struct task_struct __rcu	*real_parent;   	/* Real parent process: */
	struct task_struct __rcu	*parent;          /* Recipient of SIGCHLD, wait4() reports: */
	struct list_head		children;
	struct list_head		sibling;
	struct task_struct		*group_leader;

	struct list_head		ptraced;
	struct list_head		ptrace_entry;

	struct pid			*thread_pid;              	/* PID/PID hash table linkage. */
	struct hlist_node		pid_links[PIDTYPE_MAX];
	struct list_head		thread_group;
	struct list_head		thread_node;
	struct completion		*vfork_done;

	int __user			*set_child_tid;
	int __user			*clear_child_tid;

	u64				utime;
	u64				stime;
	u64				gtime;
	struct prev_cputime		prev_cputime;

	unsigned long			nvcsw;                   	/* Context switch counts: */
	unsigned long			nivcsw;

	u64				start_time;
	u64				start_boottime;

	unsigned long			min_flt;                 	/* MM fault and swap info */
	unsigned long			maj_flt;

	const struct cred __rcu		*ptracer_cred;
	const struct cred __rcu		*real_cred;
	const struct cred __rcu		*cred;

	char				comm[TASK_COMM_LEN];
	struct nameidata		*nameidata;

	struct sysv_sem			sysvsem;
	struct sysv_shm			sysvshm;

	struct fs_struct		*fs;                   	/* Filesystem information: */
	struct files_struct		*files;               /* Open file information: */

	struct nsproxy			*nsproxy;               /* Namespaces: */

	/* Signal handlers: */
	struct signal_struct		*signal;
	struct sighand_struct		*sighand;
	sigset_t			blocked;
	sigset_t			real_blocked;
	sigset_t			saved_sigmask;
	struct sigpending		pending;
	unsigned long			sas_ss_sp;
	size_t				sas_ss_size;
	unsigned int			sas_ss_flags;
	struct callback_head		*task_works;

	kuid_t				loginuid;
	unsigned int			sessionid;
	struct seccomp			seccomp;
	u32				parent_exec_id;
	u32				self_exec_id;

	spinlock_t			alloc_lock;     	/* Protection against (de-)allocation: mm, files, fs, tty, keyrings, mems_allowed, mempolicy: */
	raw_spinlock_t			pi_lock;      /* Protection of the PI data structures: */

	struct wake_q_node		wake_q;
	void				*journal_info;       	/* Journalling filesystem info: */
	struct bio_list			*bio_list;   	/* Stacked block device info: */

	struct reclaim_state		*reclaim_state;
	struct backing_dev_info		*backing_dev_info;
	struct io_context		*io_context;

	unsigned long			ptrace_message;
	kernel_siginfo_t		*last_siginfo;

	struct task_io_accounting	ioac;
	struct css_set __rcu		*cgroups;
	struct list_head		cg_list;
	struct robust_list_head __user	*robust_list;
	struct list_head		pi_state_list;
	struct futex_pi_state		*pi_state_cache;
	struct mutex			futex_exit_mutex;
	unsigned int			futex_state;
	struct perf_event_context	*perf_event_ctxp[perf_nr_task_contexts];
	struct mutex			perf_event_mutex;
	struct list_head		perf_event_list;

	struct mempolicy		*mempolicy;
	short				il_prev;
	short				pref_node_fork;
	int				numa_scan_seq;
	unsigned int			numa_scan_period;
	unsigned int			numa_scan_period_max;
	int				numa_preferred_nid;
	unsigned long			numa_migrate_retry;
	/* Migration stamp: */
	u64				node_stamp;
	u64				last_task_numa_placement;
	u64				last_sum_exec_runtime;
	struct callback_head		numa_work;
	struct numa_group __rcu		*numa_group;
	unsigned long			*numa_faults;
	unsigned long			total_numa_faults;
	unsigned long			numa_faults_locality[3];
	unsigned long			numa_pages_migrated;

	struct tlbflush_unmap_batch	tlb_ubc;
	union {
		refcount_t		rcu_users;
		struct rcu_head		rcu;
	};

	struct pipe_inode_info		*splice_pipe;

	struct page_frag		task_frag;
	int				nr_dirtied;
	int				nr_dirtied_pause;
	unsigned long			dirty_paused_when;

	/*
	 * Time slack values; these are used to round up poll() and
	 * select() etc timeout values. These are in nanoseconds.
	 */
	u64				timer_slack_ns;
	u64				default_timer_slack_ns;
	int				curr_ret_stack;             	/* Index of current stored address in ret_stack: */
	int				curr_ret_depth;
	struct ftrace_ret_stack		*ret_stack;   /* Stack of return addresses for return function tracing: */

	unsigned long long		ftrace_timestamp;  	/* Timestamp for last schedule: */
	atomic_t			trace_overrun;
	atomic_t			tracing_graph_pause;
	unsigned long			trace;
	unsigned long			trace_recursion;

	struct mem_cgroup		*memcg_in_oom;
	gfp_t				memcg_oom_gfp_mask;
	int				memcg_oom_order;
  unsigned int			memcg_nr_pages_over_high;      	/* Number of pages to reclaim on returning to userland: */
	struct mem_cgroup		*active_memcg;              	/* Used by memcontrol for targeted memcg charge: */
	struct request_queue		*throttle_queue;

	struct uprobe_task		*utask;
	int				pagefault_disabled;
	struct task_struct		*oom_reaper_list;

	struct vm_struct		*stack_vm_area;
	refcount_t			stack_refcount;                 	/* A live task holds one reference: */

	void				*security;                           	/* Used by LSM modules for access restriction: */

	struct thread_struct		thread;                  	/* CPU-specific state of this task: */
};

```

```
struct rq {

	struct cfs_rq		cfs;
	struct rt_rq		rt;
	struct dl_rq		dl;

	struct list_head	leaf_cfs_rq_list;
	struct list_head	*tmp_alone_branch;

	struct task_struct	*curr;
	struct task_struct	*idle;
	struct task_struct	*stop;

	struct mm_struct	*prev_mm;

	struct root_domain		*rd;
	struct sched_domain __rcu	*sd;
	struct callback_head	*balance_callback;
	struct list_head cfs_tasks;

	struct llist_head	wake_list;
```

```
struct sched_entity {
	struct load_weight		load;
	struct rb_node			run_node;
	struct list_head		group_node;

	u64				vruntime;

	struct sched_entity		*parent;
	/* rq on which this entity is (to be) queued: */
	struct cfs_rq			*cfs_rq;
	/* rq "owned" by this entity/group: */
	struct cfs_rq			*my_q;
};
```

### sched/core.c
// Default task group
`struct task_group root_task_group`

* `schedule()`
```
schedule
  preempt_disable
  __schedule
    	local_irq_disable
			rcu_note_context_switch
      rq_lock(rq, &rf)
      smp_mb__after_spinlock
      pick_next_task
				// see its section
      context_switch
        see context switch section
    repeat if TIF_NEED_RESCHED is set in current thread
  sched_preempt_enable_no_resched  
```

* `context_switch`
- kernel -> kernel   lazy + transfer active
-   user -> kernel   lazy + mmgrab() active

- kernel ->   user   switch + mmdrop() active
-   user ->   user   switch
```
context_switch
  prepare_task_switch
  arch_start_context_switch

  for different case:
    enter_lazy_tlb
    mmgrab
			atomic_inc(mm->mm_count)
    membarrier_switch_mm
    ->mm and ->active_mm
    prepare_lock_switch
    switch_to
      // arch dependent code in assembly
```

* `sched/wait.c`
`wait.h`
```
struct wait_queue_entry {
	unsigned int		flags;
	void			*private;
	wait_queue_func_t	func;
	struct list_head	entry;
};

typedef int (*wait_queue_func_t)(struct wait_queue_entry *wq_entry, unsigned mode, int flags, void *key)
```		

```
__wake_up_sync_key
	__wake_up_common_lock
		init wait_queue_entry_t bookmark
		spin_lock_irqsave(&wq_head->lock, flags)
		__wake_up_common
			curr = list_first_entry(&wq_head->head, wait_queue_entry_t, entry)
			list_for_each_entry_safe_from(curr, next, &wq_head->head, entry)
				curr->func(curr, mode, wake_flags, key)

```

```
switch_to
	// arch/x86/entry/entry_64.S
	__switch_to_asm
		save callee-saved registers as ABI
		copy to the target %rsp from %rdi the second parameter
		jump to __switch_to() /arch/x86/kernel/process_64.c
			save_fsgs
			load_TLS
			arch_end_context_switch

			savesegment(es, prev->es)
			savesegment(ds, prev->ds)

			update_task_stack(next_p)
			resctrl_sched_in()
```


```
pick_next_task
	// optimization: jump to CFS or IDLE class directly

	// balance the tasks from the prev task's class
		clss->balance()

	for_each_class
		 class->pick_next_task
```


### kernel/time/timer.c
- timer triggered scheduling
```
tick_periodic tick-common.c / tick_sched_handle	tick-sched.c
	update_process_times
		run_local_timers
		rcu_sched_clock_irq

		scheduler_tick
			// see its section
```

```
sched_clock_tick
	sched_clock_tick

	rq->curr->sched_class->task_tick

	trigger_load_balance
```
