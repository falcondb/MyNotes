
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