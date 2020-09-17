### BPF loader in tools/perf/util/bpf-loader.h/
```


```


### BPF Syscalls in kernel/bpf/syscall.c ###

* SYSCALL
```
// SYSCALL_DEFINEx in include/linux/syscalls.h
SYSCALL_DEFINE3(bpf, int, cmd, union bpf_attr __user *, uattr, unsigned int, size) {
  copy_from_user()
  security_bpf() \\ TODO: trace down the code
  switch (cmd) {
    case BPF_MAP_CREATE:
    err = map_create(&attr);
    ...
  }

}
```
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
  /* check attr->attach_type to find the internal bpf_prog_type

  bpf_prog_get_type(attr->attach_bpf_fd, ptype);

  sock_map_get_from_fd
  or cgroup_bpf_prog_attach
  or lirc_prog_attach
  or skb_flow_dissector_bpf_prog_attach

```
* sock_map_get_from_fd
```
/* include/linux/bpf.h, net/core/sock_map.c
sock_map_get_from_fd() ==> sock_map_prog_update() {
  // include/uapi/linux/bpf.h
  case BPF_SK_MSG_VERDICT:
  /* include/liinux/skmsg.h psock_set_prog ==> xchg (e.g., asm-generic/atomic-instrumented.h)
    xchg ==> kasan_check_write;
             arch_xchg ==> __xchg_wrapper (arch/arm64/include/asm/cmpxchg.h) ==>
             __xchg_case_##name##sz : asm code

    struct sk_psock_progs {
     	struct bpf_prog			*msg_parser;
     	struct bpf_prog			*skb_parser;
     	struct bpf_prog			*skb_verdict;
     };  
  */
  psock_set_prog(&progs->msg_parser, prog);
case BPF_SK_SKB_STREAM_PARSER:
  psock_set_prog(&progs->skb_parser, prog);
case BPF_SK_SKB_STREAM_VERDICT:
  psock_set_prog(&progs->skb_verdict, prog);
}
```

* cgroup_bpf_prog_attach in kernel/bpf/cgroup
```
cgroup_bpf_prog_attach(const union bpf_attr *attr,
			   enum bpf_prog_type ptype, struct bpf_prog *prog) {
  cgroup_get_from_fd(attr->target_fd);
  cgroup_bpf_attach()
}


/* cgroup_bpf_attach(): __cgroup_bpf_*() protected by cgroup_mutex in cgroup/cgroup.h */
__cgroup_bpf_attach() {
  int __cgroup_bpf_attach(struct cgroup *cgrp, struct bpf_prog *prog,
			enum bpf_attach_type type, u32 flags)
{
	for_each_cgroup_storage_type(stype) {
		storage[stype] = bpf_cgroup_storage_alloc(prog, stype);
	}

	if (flags & BPF_F_ALLOW_MULTI) {
    ...
	} else {
		if (list_empty(progs)) {
			kmalloc(sizeof(*pl), GFP_KERNEL);
			list_add_tail(&pl->node, progs);
		} else {
			pl = list_first_entry(progs, typeof(*pl), node);
			old_prog = pl->prog;
			for_each_cgroup_storage_type(stype) {
				old_storage[stype] = pl->storage[stype];
				bpf_cgroup_storage_unlink(old_storage[stype]);
			}
		}
		pl->prog = prog;
		for_each_cgroup_storage_type(stype)
			pl->storage[stype] = storage[stype];
	}

	cgrp->bpf.flags[type] = flags;

	update_effective_progs(cgrp, type);

	static_branch_inc(&cgroup_bpf_enabled_key);
	for_each_cgroup_storage_type(stype) {
		if (!old_storage[stype])
			continue;
		bpf_cgroup_storage_free(old_storage[stype]);
	}
	if (old_prog) {
		bpf_prog_put(old_prog);
		static_branch_dec(&cgroup_bpf_enabled_key);
	}
	for_each_cgroup_storage_type(stype)
		bpf_cgroup_storage_link(storage[stype], cgrp, type);
	return 0;
}      
  }
```

* BPF Prog Run
```
BPF_PROG_RUN_ARRAY ==> __BPF_PROG_RUN_ARRAY
_item = &_array->items[0];
while ((_prog = READ_ONCE(_item->prog))) {		\
  bpf_cgroup_storage_set(_item->cgroup_storage);	\
  _ret &= func(_prog, ctx);	\
  _item++;			\
}

#define BPF_PROG_RUN(prog, ctx)	({				\
 (*(prog)->bpf_func)(ctx, (prog)->insnsi);	\
	})
```

```
__bpf_trace_run  ==>  BPF_PROG_RUN  ==>  (*(prog)->bpf_func)(ctx, (prog)->insnsi);
```
