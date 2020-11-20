* net filter
```
// net/core/filter.c
BPF_CALL_1(bpf_skb_get_pay_offset, struct sk_buff *, skb)
{
	return skb_get_poff(skb);
}

// net/core/flow_dissector.c
skb_get_poff == > __skb_get_poff {
  	u32 poff = keys->control.thoff;
  // skip IP fragments
  switch (keys->basic.ip_proto) {
		poff += sizeof(struct udphdr);
    ...
}
```

* kprobes https://lwn.net/Articles/132196/
```
struct kprobe {
	struct hlist_node hlist;

	/* list of kprobes for multi-handler support */
	struct list_head list;

	/*count the number of times this probe was temporarily disarmed */
	unsigned long nmissed;

	/* location of the probe point */
	kprobe_opcode_t *addr;

	/* Allow user to indicate symbol name of the probe point */
	const char *symbol_name;

	/* Offset into the symbol */
	unsigned int offset;

	/* Called before addr is executed. */
	kprobe_pre_handler_t pre_handler;

	/* Called after addr is executed, unless... */
	kprobe_post_handler_t post_handler;

	/*
	 * ... called if executing addr causes a fault (eg. page fault).
	 * Return 1 if it handled fault, otherwise kernel will see it.
	 */
	kprobe_fault_handler_t fault_handler;

	/* Saved opcode (which has been replaced with breakpoint) */
	kprobe_opcode_t opcode;

	/* copy of the original instruction */
	struct arch_specific_insn ainsn;

	/*
	 * Indicates various status flags.
	 * Protected by kprobe_mutex after this kprobe is registered.
	 */
	u32 flags;
};
enable_kprobe ==> __arm_kprobe ==> get_optimized_kprobe()
==> arch_arm_kprobe(); optimize_kprobe(); get_optimized_kprobe()
==> /* arm64 */ patch_text(p->addr, BRK64_OPCODE_KPROBES);
optimize_kprobe() ==> kick_kprobe_optimizer() ==> schedule_delayed_work() ???
==> do_optimize_kprobes() ==> arch_optimize_kprobes()


// kernel/kprobes.c
register_kprobe() {
  	old_p = get_kprobe(p->addr);   // search kprobe_table hashtable for probe list
    prepare_kprobe(p);
    hlist_add_head_rcu
    arm_kprobe
    try_to_optimize_kprobe
}

register_kretprobe() {
  pre_handler_kretprobe() ==> arch_prepare_kretprobe; hlist_add_head(&ri->hlist, &kretprobe_inst_table[hash]);
}
```

* include/linux/trace_events.h
```
struct trace_entry {
	unsigned short		type;
	unsigned char		flags;
	unsigned char		preempt_count;
	int			pid;
};

/*
 * Trace iterator - used by printout routines who present trace
 * results to users and which routines might sleep, etc:
 */
struct trace_iterator {
};

struct trace_event {
	struct hlist_node		node;
	struct list_head		list;
	int				type;
	struct trace_event_functions	*funcs;
};

struct trace_event_class {
	const char		*system;
	void			*probe;
	void			*perf_probe;
  struct list_head	*(*get_fields)(struct trace_event_call *);
  struct list_head	fields;
	int			(*reg)(struct trace_event_call *event, enum trace_reg type, void *data);
	int			(*define_fields)(struct trace_event_call *);
	int			(*raw_init)(struct trace_event_call *);
};

struct trace_event_call {
	struct list_head	list;
	struct trace_event_class *class;
	union {
		char			*name;
		/* Set TRACE_EVENT_FL_TRACEPOINT flag when using "tp" */
		struct tracepoint	*tp;
	};
	struct trace_event	event;
	char			*print_fmt;
	struct event_filter	*filter;
	void			*mod;
	void			*data;
	int			  flags; /* static flags of different events */
	int				perf_refcount;
	struct hlist_head __percpu	*perf_events;
	struct bpf_prog_array __rcu	*prog_array;
	int	(*perf_perm)(struct trace_event_call *, struct perf_event *);
};

struct trace_event_file {
	struct list_head		list;
	struct trace_event_call		*event_call;
	struct event_filter __rcu	*filter;
	struct dentry			*dir;
	struct trace_array		*tr;
	struct trace_subsystem_dir	*system;
	struct list_head		triggers;
	unsigned long		flags;
	atomic_t		sm_ref;	/* soft-mode reference counter */
	atomic_t		tm_ref;	/* trigger-mode reference counter */
};

#define event_trace_printk(ip, fmt, args...)				\
do {									\
	__trace_printk_check_format(fmt, ##args);			\
	tracing_record_cmdline(current);				\
	if (__builtin_constant_p(fmt)) {				\
		static const char *trace_printk_fmt			\
		  __attribute__((section("__trace_printk_fmt"))) =	\
			__builtin_constant_p(fmt) ? fmt : NULL;		\
									\
		__trace_bprintk(ip, trace_printk_fmt, ##args);		\
	} else								\
		__trace_printk(ip, fmt, ##args);			\
} while (0)

```
* bpf_trace
```

static struct bpf_raw_event_map *bpf_get_raw_tracepoint_module(const char *name)
{
	struct bpf_raw_event_map *btp, *ret = NULL;
	struct bpf_trace_module *btm;
	unsigned int i;

	mutex_lock(&bpf_module_mutex);
	list_for_each_entry(btm, &bpf_trace_modules, list) {
		for (i = 0; i < btm->module->num_bpf_raw_events; ++i) {
			btp = &btm->module->bpf_raw_events[i];
			if (!strcmp(btp->tp->name, name)) {
				if (try_module_get(btm->module))
					ret = btp;
				goto out;
			}
		}
	}
out:
	mutex_unlock(&bpf_module_mutex);
	return ret;
}





int perf_event_attach_bpf_prog(struct perf_event *event, struct bpf_prog *prog){
	old_array = bpf_event_rcu_dereference(event->tp_event->prog_array);
  bpf_prog_array_copy(old_array, NULL, prog, &new_array);
	/* set the new array to event->tp_event and set event->prog */
	event->prog = prog;
	rcu_assign_pointer(event->tp_event->prog_array, new_array);
}


int perf_event_query_prog_array(struct perf_event *event, void __user *info){
	progs = bpf_event_rcu_dereference(event->tp_event->prog_array);
	bpf_prog_array_copy_info(progs, ids, ids_len, &prog_cnt);

	copy_to_user(&uquery->prog_cnt, &prog_cnt, sizeof(prog_cnt))
	copy_to_user(uquery->ids, ids, ids_len * sizeof(u32)))
}


void __bpf_trace_run(struct bpf_prog *prog, u64 *args)
{
	rcu_read_lock();
	preempt_disable();
	(void) BPF_PROG_RUN(prog, args);
	preempt_enable();
	rcu_read_unlock();
}


int __bpf_probe_register(struct bpf_raw_event_map *btp, struct bpf_prog *prog)
{
	struct tracepoint *tp = btp->tp;
	tracepoint_probe_register(tp, (void *)btp->bpf_func, prog);  
  ==> tracepoint_probe_register_prio  ==> tracepoint_add_func ==>
}


static int bpf_event_notify(struct notifier_block *nb, unsigned long op, void *module)
{
	switch (op) {
	case MODULE_STATE_COMING:
		btm = kzalloc(sizeof(*btm), GFP_KERNEL);
		list_add(&btm->list, &bpf_trace_modules);
	case MODULE_STATE_GOING:
		list_for_each_entry_safe(btm, tmp, &bpf_trace_modules, list) {
			if (btm->module == module) {
				list_del(&btm->list);
				kfree(btm);
        }
}

static struct notifier_block bpf_module_nb = {
	.notifier_call = bpf_event_notify,
};

static int __init bpf_event_init(void)
{
	register_module_notifier(&bpf_module_nb);
	return 0;
}


```
* trace_kprobe
```
/* Event call and class holder */
struct trace_probe_event {
	unsigned int			flags;	/* For TP_FLAG_* */
	struct trace_event_class	class;
	struct trace_event_call		call;
	struct list_head 		files;
	struct list_head		probes;
};

struct trace_probe {
	struct list_head		list;
	struct trace_probe_event	*event;
	ssize_t				size;	/* trace entry size */
	unsigned int			nr_args;
	struct probe_arg		args[];
};

struct event_file_link {
	struct trace_event_file		*file;
	struct list_head		list;
};

struct trace_kprobe {
	struct dyn_event	devent;
	struct kretprobe	rp;	/* Use rp.kp for kprobe use */
	unsigned long __percpu *nhit;
	const char		*symbol;	/* symbol name */
	struct trace_probe	tp;
};

// trace_dynevent.h
struct dyn_event_operations trace_kprobe_ops = {
	.create = trace_kprobe_create,
	.show = trace_kprobe_show,
	.is_busy = trace_kprobe_is_busy,
	.free = trace_kprobe_release,
	.match = trace_kprobe_match,
};

__enable_trace_kprobe {
  enable_kprobe(&tk->rp.kp); || enable_kretprobe(&tk->rp);
}

enable_trace_kprobe(struct trace_event_call *call, struct trace_event_file *file) {
  tp = trace_probe_primary_from_call(call);
  trace_probe_add_file(tp, file); || trace_probe_set_flag(tp, TP_FLAG_PROFILE);
  for each in trace_probe_probe_list(tp)
    __enable_trace_kprobe
}

register_trace_kprobe {
  register_kprobe_event
  __register_trace_kprobe
  dyn_event_add
}

/* Kprobe handler */
file_operations kprobe_events_ops {.write = probes_write} ==>
probes_write ==> create_or_delete_trace_kprobe ==>
trace_kprobe_create ==> alloc_trace_kprobe ==> kprobe_dispatcher ==>
kprobe_perf_func(struct trace_kprobe *tk, struct pt_regs *regs)
{
  if (bpf_prog_array_valid(call)) {
    trace_call_bpf(call, regs); ==> BPF_PROG_RUN_ARRAY ==> __BPF_PROG_RUN_ARRAY

    /*
     * We need to check and see if we modified the pc of the
     * pt_regs, and if so return 1 so that we don't do the
     * single stepping.
     */
  }
  head = this_cpu_ptr(call->perf_events);

  entry = perf_trace_buf_alloc(size, NULL, &rctx);

  entry->ip = (unsigned long)tk->rp.kp.addr;
  store_trace_args(&entry[1], &tk->tp, regs, sizeof(*entry), dsize);
  perf_trace_buf_submit(entry, size, rctx, call->event.type, 1, regs, head, NULL);
}

__kprobe_trace_func {
  trace_event_buffer_lock_reserve

  entry = ring_buffer_event_data(event);
  entry->ip = (unsigned long)tk->rp.kp.addr;
  store_trace_args(&entry[1], &tk->tp, regs, sizeof(*entry), dsize);
  event_trigger_unlock_commit_regs(trace_file, buffer, event, entry, irq_flags, pc, regs);
}

kprobe_event_define_fields {

}
```
* tracepoint-defs.h
```
struct tracepoint_func {
	void *func;
	void *data;
	int prio;
};

struct tracepoint {
	const char *name;		/* Tracepoint name */
	struct static_key key;
	int (*regfunc)(void);
	void (*unregfunc)(void);
	struct tracepoint_func __rcu *funcs;
};

struct bpf_raw_event_map {
	struct tracepoint	*tp;
	void			*bpf_func;
	u32			num_args;
	u32			writable_size;
} __aligned(32);
```

* kernel/events/core
```

```

* arch/x86/kernel/kprobes/ftrace.c
```
kprobe_ftrace_handler
```
