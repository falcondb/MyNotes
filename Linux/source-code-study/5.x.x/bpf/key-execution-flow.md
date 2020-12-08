### BPF loader in tools/perf/util/bpf-loader.h/
```


```


### BPF Syscalls in kernel/bpf/syscall.c ###

* SYSCALL
  * syscalls for Map operations (create, delete, query, etc.)
  * BPF object operations
  * program operations (load, attach, detach, etc.)
  * fd operations
  * raw tracepoint
  * BTF operations.

* bpf_prog_load
```
 bpf_prog_load(union bpf_attr *attr, union bpf_attr __user *uattr)
 static int bpf_prog_load(union bpf_attr *attr, union bpf_attr __user *uattr)
{
	copy_from_user(prog->insns, u64_to_user_ptr(attr->insns),bpf_prog_insn_size(prog))

  /* if offload to dev is needed
  bpf_prog_offload_init(prog, attr);

	/* find program type: socket_filter vs tracing_filter */
	find_prog_type(type, prog);

	/* run eBPF verifier */
	bpf_check(&prog, attr, uattr);
    // see notes below
	prog = bpf_prog_select_runtime(prog, &err);

	bpf_prog_alloc_id(prog);

	/* Upon success of bpf_prog_alloc_id(), the BPF prog is
	 * effectively publicly exposed. However, retrieving via
	 * bpf_prog_get_fd_by_id() will take another reference,
	 * therefore it cannot be gone underneath us.
	 *
	 * Only for the time /after/ successful bpf_prog_new_fd()
	 * and before returning to userspace, we might just hold
	 * one reference and any parallel close on that fd could
	 * rip everything out. Hence, below notifications must
	 * happen before bpf_prog_new_fd().
	 *
	 * Also, any failure handling from this point onwards must
	 * be using bpf_prog_put() given the program is exposed.
	 */
	bpf_prog_kallsyms_add(prog);   // added to bpf_prog_aux.ksym_lnode/tnode
	perf_event_bpf_event(prog, PERF_BPF_EVENT_PROG_LOAD, 0);

	bpf_prog_new_fd(prog);   // anon_inode_getfd("bpf-map", &bpf_map_fops, map, ...);

}
```

* bpf_prog_attach
```
SYSCALL_DEFINE3(bpf
  bpf_prog_attach
    /* check attr->attach_type to find the internal bpf_prog_type
    bpf_prog_get_type(attr->attach_bpf_fd, ptype);
    sock_map_get_from_fd   // for BPF_PROG_TYPE_SK_SKB or BPF_PROG_TYPE_SK_MSG
    or cgroup_bpf_prog_attach

```
* sock_map_get_from_fd
```
/* include/linux/bpf.h, net/core/sock_map.c
sock_map_get_from_fd ==> sock_map_prog_update
  struct sk_psock_progs *progs = sock_map_progs(map)
  reset the target struct bpf_prog in sk_psock_progs
}
```

* cgroup_bpf_prog_attach in kernel/bpf/cgroup
```
cgroup_bpf_prog_attach
  cgroup_get_from_fd
  cgroup_bpf_attach ==> mutex_lock  __cgroup_bpf_attach
    bpf_cgroup_storage_alloc
      kmalloc for struct bpf_cgroup_storage and BPF map
      kmalloc for struct bpf_prog_list   list_add_tail
    update_effective_progs
      	/* allocate and recompute effective prog arrays */
        css_for_each_descendant_pre //struct cgroup_subsys_state
          compute_effective_progs
        /* all allocations were successful. Activate all prog arrays */
  	    css_for_each_descendant_pre
          activate_effective_progs
            rcu_replace_pointer
    bpf_cgroup_storage_link
      update struct bpf_cgroup_storage
```

* BPF Prog Run
```
# linux/filter.h
BPF_PROG_RUN(prog, ctx)   //  struct bpf_pro prog
 (*(prog)->bpf_func)(ctx, (prog)->insnsi)

# linux/bpf.h
BPF_PROG_RUN_ARRAY  ==> __BPF_PROG_RUN_ARRAY(array, ctx, func, false) // func = BPF_PROG_RUN
  preempt_disable()
  rcu_read_lock()
  for each _item->prog  ==>  func(_prog, ctx)  
```

```
__bpf_trace_run  ==>  BPF_PROG_RUN  ==>  (*(prog)->bpf_func)(ctx, (prog)->insnsi);
```

* perf_event_bpf_event in kernel/events/core.c
```
perf_event_bpf_event()
  PERF_BPF_EVENT_PROG_LOAD || PERF_BPF_EVENT_PROG_UNLOAD ==>
      perf_event_bpf_emit_ksymbols ==>
        perf_event_ksymbol(PERF_RECORD_KSYMBOL_TYPE_BPF, ...)
  perf_iterate_sb  
```

### BPF helpers
Each helper implementation is registered at its _struct bpf_func_proto_
```
#linux/bpf.h
struct bpf_func_proto {
	u64 (*func)(u64 r1, u64 r2, u64 r3, u64 r4, u64 r5);
	bool gpl_only, pkt_access;
	enum bpf_return_type ret_type;
	union {
		struct {
			enum bpf_arg_type arg1_type;
			enum bpf_arg_type arg2_type;
			enum bpf_arg_type arg3_type;
			enum bpf_arg_type arg4_type;
			enum bpf_arg_type arg5_type;
		};
		enum bpf_arg_type arg_type[5];
	};
	int *btf_id; /* BTF ids of arguments */
};
```

kprobe_prog_func_proto          kprobe_verifier_ops
tp_prog_func_proto              tracepoint_verifier_ops
pe_prog_func_proto              perf_event_verifier_ops
raw_tp_prog_func_proto          raw_tracepoint_verifier_ops
tracing_prog_func_proto         tracing_verifier_ops
tracing_func_proto
bpf_get_probe_write_proto
bpf_get_trace_printk_proto

```
struct bpf_verifier_ops {
	/* return eBPF function prototype for verification */
	const struct bpf_func_proto *
	(*get_func_proto)(enum bpf_func_id func_id, const struct bpf_prog *prog);
	bool (*is_valid_access)(...);
	int (*gen_prologue)(...);
	int (*gen_ld_abs)(...);
	u32 (*convert_ctx_access)(...);
};
```

veificer ops are registered as
```
static const struct bpf_verifier_ops * const bpf_verifier_ops[] = {
#define BPF_PROG_TYPE(_id, _name, prog_ctx_type, kern_ctx_type) \
	[_id] = & _name ## _verifier_ops,
#define BPF_MAP_TYPE(_id, _ops)
#include <linux/bpf_types.h>
}
```

check_helper_call TO BE STUDIED
```
#kernel/bpf/verifier.c
bpf_prog_load
  bpf_check
    do_check
      check_helper_call
```
