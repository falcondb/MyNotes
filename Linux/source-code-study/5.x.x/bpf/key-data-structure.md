
# Key data structures and function prototypes #

### include/uapi/linux/bpf_common.h ###
* #define BPF_CLASS(code) ((code) & 0x07)		0x00 // https://github.com/iovisor/bpf-docs/blob/master/eBPF.md
* #define BPF_SIZE(code)  ((code) & 0x18)
* #define BPF_MODE(code)  ((code) & 0xe0)
* #define BPF_OP(code)    ((code) & 0xf0)



### /include/linux/filter.h ###
* BPF_REG_ARG1   BPF_REG_1  etc.    // register definitions

* #define BPF_ALU64_REG(OP, DST, SRC)	((struct bpf_insn) {}  etc.

* BPF_CALL_x
```#define __BPF_MAP(n, ...) __BPF_MAP_##n(__VA_ARGS__)
 #define __BPF_MAP_{0 ... 5}
 #define __BPF_REG(n, ...) __BPF_REG_##n(__VA_ARGS__)
 #define BPF_CALL_0 -->
  #define BPF_CALL_x(x, name, ...)	   -->
	u64 ____##name(__BPF_MAP(x, __BPF_DECL_ARGS, __BPF_V, __VA_ARGS__));
  u64 name(__BPF_REG(x, __BPF_DECL_REGS, __BPF_N, __VA_ARGS__))	       \
  {								       \
    return ((btf_##name)____##name)(__BPF_MAP(x,__BPF_CAST,__BPF_N,__VA_ARGS__));\
  }
```
 // TODO: not fully understand how the helper function prototypes are generated

* struct bpf_prog               // for kernel
```
struct bpf_prog {
	u16			pages;		/* Number of allocated pages */
	u16  // different flags 			
	enum bpf_attach_type	expected_attach_type;
	u32			len;		/* Number of filter blocks */
	u32			jited_len;	/* Size of jited insns in bytes */
	u8			tag[BPF_TAG_SIZE];
	struct bpf_prog_aux	*aux;		/* Auxiliary fields */
	struct sock_fprog_kern	*orig_prog;	/* Original BPF program */
	unsigned int (*bpf_func)(const void *ctx, const struct bpf_insn *insn);
	/* Instructions for interpreter */
	union {
		struct sock_filter	insns[0];
		struct bpf_insn		insnsi[0];
	};
};
```


### include/uapi/inux/filter.h ###
* struct sock_filter        // BST cBPF filter


### /include/linux/bpf.h ###
* struct bpf_map_ops        // function pointers for map operations
* bpf_map_memory  { u32 pages, user_struct *user}
* struct bpf_map            // the internal presentation of bpf map
* extern const struct file_operations bpf_map_fops;
* extern const struct file_operations bpf_prog_fops;

* enum bpf_arg_type         // function argument constraints
* enum bpf_return_type
* struct bpf_func_proto     // prototype of in-kernel helper functions
* enum bpf_reg_type         // types of values stored in eBPF registers
* struct bpf_verifier_ops   { bpf_func_proto, is_valid_access,
                            gen_prologue, gen_ld_abs, convert_ctx_access }
* struct bpf_prog_aux
* struct bpf_array
* struct bpf_event_entry
* struct bpf_prog_array
* #define __BPF_PROG_RUN_ARRAY

### linux/bpf.h
```
struct bpf_map {
  const struct bpf_map_ops *ops ____cacheline_aligned;
  struct bpf_map *inner_map_meta;
  enum bpf_map_type map_type;
  u32 key_size;
  u32 value_size;
  u32 max_entries;
  u32 map_flags;
  int spin_lock_off; /* >=0 valid offset, <0 error */
  u32 id;
  int numa_node;
  u32 btf_key_type_id, btf_value_type_id;
  struct btf *btf;
  struct bpf_map_memory memory;
  char name[BPF_OBJ_NAME_LEN];
  bool unpriv_array;
  bool frozen; /* write-once; write-protected by freeze_mutex */
  /* 22 bytes hole */
  atomic64_t refcnt ____cacheline_aligned;
  atomic64_t usercnt;
  struct work_struct work;
  struct mutex freeze_mutex;
  u64 writecnt; /* writable mmap cnt; protected by freeze_mutex */
};

struct bpf_map_ops {
	/* funcs callable from userspace (via syscall) */
	int (*map_alloc_check)(union bpf_attr *attr);
	struct bpf_map *(*map_alloc)(union bpf_attr *attr);
	void (*map_release)(struct bpf_map *map, struct file *map_file);
	void (*map_free)(struct bpf_map *map);
	int (*map_get_next_key)(struct bpf_map *map, void *key, void *next_key);
	void (*map_release_uref)(struct bpf_map *map);
	void *(*map_lookup_elem_sys_only)(struct bpf_map *map, void *key);

  /* funcs callable from userspace and from eBPF programs */
	void *(*map_lookup_elem)(struct bpf_map *map, void *key);
	int (*map_update_elem)(struct bpf_map *map, void *key, void *value, u64 flags);
	int (*map_delete_elem)(struct bpf_map *map, void *key);
	int (*map_push_elem)(struct bpf_map *map, void *value, u64 flags);
	int (*map_pop_elem)(struct bpf_map *map, void *value);
	int (*map_peek_elem)(struct bpf_map *map, void *value);

	/* funcs called by prog_array and perf_event_array map */
	void *(*map_fd_get_ptr)(struct bpf_map *map, struct file *map_file, int fd);
	void (*map_fd_put_ptr)(void *ptr);
	u32 (*map_gen_lookup)(struct bpf_map *map, struct bpf_insn *insn_buf);
	u32 (*map_fd_sys_lookup_elem)(void *ptr);
	void (*map_seq_show_elem)(struct bpf_map *map, void *key, struct seq_file *m);
	int (*map_check_btf)(...);

	/* Prog poke tracking helpers. */
	int (*map_poke_track)(struct bpf_map *map, struct bpf_prog_aux *aux);
	void (*map_poke_untrack)(struct bpf_map *map, struct bpf_prog_aux *aux);
	void (*map_poke_run)(...);

	/* Direct value access helpers. */
	int (*map_direct_value_addr)(...);
	int (*map_direct_value_meta)(...);
	int (*map_mmap)(...);
};
```

### /include/uapi/linux/bpf.h ###
* BPF Register numbers
```
enum {
	BPF_REG_0 = 0,
  ...,
  __MAX_BPF_REG,
  }
```  
* BPF_ALU64                // instruction classes
* bpf_insn
```
struct bpf_insn {
  code,
  dst_rg:4,
  src_reg:4,
  off,
  imm
  } // BPF instruction
```  
* enum bpf_cmd
* enum bpf_map_type
* enum bpf_prog_type
* enum bpf_attach_type
* union bpf_attr
* enum bpf_ret_code
* struct bpf_prog_info
* struct bpf_map_info
* struct bpf_btf_info
* struct bpf_stack_build_id
* struct bpf_sock_addr      // another bpf_sock data structure?? TODO: usage of it
* struct bpf_sock_ops
* struct bpf_fib_lookup

* #define __BPF_FUNC_MAPPER(FN)
* struct __sk_buff          // user accessible mirror of in-kernel sk_buff
* struct bpf_sock
* struct bpf_tcp_sock
* struct bpf_sock_tuple

* enum xdp_action
* struct xdp_md
* struct sk_msg_md
* struct sk_reuseport_md

### arch/arm64/net/bpf_jit.h



### kernel/bpf/helpers.c ###

### kernel/bpf/syscall.c ###


### /kernel/bpf/disasm.c ###
* __BPF_FUNC_MAPPER

### include/trace/bpf_probe.h ###

### include/linux/bpf_trace.h --> trace/events/xdp.h ###
### tools/testing/selftests/bpf/bpf_trace_helper.h ###

### kernel/trace/bpf_trace.c ###
