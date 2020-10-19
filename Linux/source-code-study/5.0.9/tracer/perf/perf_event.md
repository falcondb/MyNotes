* perf_event.h
```
struct perf_event {
#ifdef CONFIG_PERF_EVENTS
	/*
	 * entry onto perf_event_context::event_list;
	 *   modifications require ctx->lock
	 *   RCU safe iterations.
	 */
	struct list_head		event_entry;

	/*
	 * Locked for modification by both ctx->mutex and ctx->lock; holding
	 * either sufficies for read.
	 */
	struct list_head		sibling_list;
	struct list_head		active_list;
	/*
	 * Node on the pinned or flexible tree located at the event context;
	 */
	struct rb_node			group_node;
	u64				group_index;
	/*
	 * We need storage to track the entries in perf_pmu_migrate_context; we
	 * cannot use the event_entry because of RCU and we want to keep the
	 * group in tact which avoids us using the other two entries.
	 */
	struct list_head		migrate_entry;

	struct hlist_node		hlist_entry;
	struct list_head		active_entry;
	int				nr_siblings;

	/* Not serialized. Only written during event initialization. */
	int				event_caps;
	/* The cumulative AND of all event_caps for events in this group. */
	int				group_caps;

	struct perf_event		*group_leader;
	struct pmu			*pmu;
	void				*pmu_private;

	enum perf_event_state		state;
	unsigned int			attach_state;
	local64_t			count;
	atomic64_t			child_count;

	/*
	 * These are the total time in nanoseconds that the event
	 * has been enabled (i.e. eligible to run, and the task has
	 * been scheduled in, if this is a per-task event)
	 * and running (scheduled onto the CPU), respectively.
	 */
	u64				total_time_enabled;
	u64				total_time_running;
	u64				tstamp;

	/*
	 * timestamp shadows the actual context timing but it can
	 * be safely used in NMI interrupt context. It reflects the
	 * context time as it was when the event was last scheduled in.
	 *
	 * ctx_time already accounts for ctx->timestamp. Therefore to
	 * compute ctx_time for a sample, simply add perf_clock().
	 */
	u64				shadow_ctx_time;

	struct perf_event_attr		attr;
	u16				header_size;
	u16				id_header_size;
	u16				read_size;
	struct hw_perf_event		hw;

	struct perf_event_context	*ctx;
	atomic_long_t			refcount;

	/*
	 * These accumulate total time (in nanoseconds) that children
	 * events have been enabled and running, respectively.
	 */
	atomic64_t			child_total_time_enabled;
	atomic64_t			child_total_time_running;

	/*
	 * Protect attach/detach and child_list:
	 */
	struct mutex			child_mutex;
	struct list_head		child_list;
	struct perf_event		*parent;

	int				oncpu;
	int				cpu;

	struct list_head		owner_entry;
	struct task_struct		*owner;

	/* mmap bits */
	struct mutex			mmap_mutex;
	atomic_t			mmap_count;

	struct ring_buffer		*rb;
	struct list_head		rb_entry;
	unsigned long			rcu_batches;
	int				rcu_pending;

	/* poll related */
	wait_queue_head_t		waitq;
	struct fasync_struct		*fasync;

	/* delayed work for NMIs and such */
	int				pending_wakeup;
	int				pending_kill;
	int				pending_disable;
	struct irq_work			pending;

	atomic_t			event_limit;

	/* address range filters */
	struct perf_addr_filters_head	addr_filters;
	/* vma address array for file-based filders */
	unsigned long			*addr_filters_offs;
	unsigned long			addr_filters_gen;

	void (*destroy)(struct perf_event *);
	struct rcu_head			rcu_head;

	struct pid_namespace		*ns;
	u64				id;

	u64				(*clock)(void);
	perf_overflow_handler_t		overflow_handler;
	void				*overflow_handler_context;
#ifdef CONFIG_BPF_SYSCALL
	perf_overflow_handler_t		orig_overflow_handler;
	struct bpf_prog			*prog;
#endif

#ifdef CONFIG_EVENT_TRACING
	struct trace_event_call		*tp_event;
	struct event_filter		*filter;
#ifdef CONFIG_FUNCTION_TRACER
	struct ftrace_ops               ftrace_ops;
#endif
#endif

#ifdef CONFIG_CGROUP_PERF
	struct perf_cgroup		*cgrp; /* cgroup event is attach to */
#endif

	struct list_head		sb_list;
#endif /* CONFIG_PERF_EVENTS */
};
```
