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

* bpf_trace
```

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

```

* arch/x86/kernel/kprobes/ftrace.c
```
kprobe_ftrace_handler
```
