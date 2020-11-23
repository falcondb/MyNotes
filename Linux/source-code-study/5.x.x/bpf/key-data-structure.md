
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
* struct
```
struct bpf_prog_info {
	__u32 type;
	__u32 id;
	__u8  tag[BPF_TAG_SIZE];
	__u32 jited_prog_len;
	__u32 xlated_prog_len;
	__aligned_u64 jited_prog_insns;
	__aligned_u64 xlated_prog_insns;
	__u64 load_time;	/* ns since boottime */
	__u32 created_by_uid;
	__u32 nr_map_ids;
	__aligned_u64 map_ids;
	char name[BPF_OBJ_NAME_LEN];
	__u32 ifindex;
	__u32 gpl_compatible:1;
	__u32 :31; /* alignment pad */
	__u64 netns_dev;
	__u64 netns_ino;
	__u32 nr_jited_ksyms;
	__u32 nr_jited_func_lens;
	__aligned_u64 jited_ksyms;
	__aligned_u64 jited_func_lens;
	__u32 btf_id;
	__u32 func_info_rec_size;
	__aligned_u64 func_info;
	__u32 nr_func_info;
	__u32 nr_line_info;
	__aligned_u64 line_info;
	__aligned_u64 jited_line_info;
	__u32 nr_jited_line_info;
	__u32 line_info_rec_size;
	__u32 jited_line_info_rec_size;
	__u32 nr_prog_tags;
	__aligned_u64 prog_tags;
	__u64 run_time_ns;
	__u64 run_cnt;
}
```
* struct bpf_map_info
```
struct bpf_map_info {
	__u32 type;
	__u32 id;
	__u32 key_size;
	__u32 value_size;
	__u32 max_entries;
	__u32 map_flags;
	char  name[BPF_OBJ_NAME_LEN];
	__u32 ifindex;
	__u32 btf_vmlinux_value_type_id;
	__u64 netns_dev;
	__u64 netns_ino;
	__u32 btf_id;
	__u32 btf_key_type_id;
	__u32 btf_value_type_id;
}
```
* struct bpf_btf_info
* struct bpf_stack_build_id
* struct bpf_sock_addr
```
struct bpf_sock_addr {
	__u32 user_family;	/* Allows 4-byte read, but no write. */
	__u32 user_ip4;		/* Allows 1,2,4-byte read and 4-byte write.
				 * Stored in network byte order.
				 */
	__u32 user_ip6[4];	/* Allows 1,2,4,8-byte read and 4,8-byte write.
				 * Stored in network byte order.
				 */
	__u32 user_port;	/* Allows 1,2,4-byte read and 4-byte write.
				 * Stored in network byte order
				 */
	__u32 family;		/* Allows 4-byte read, but no write */
	__u32 type;		/* Allows 4-byte read, but no write */
	__u32 protocol;		/* Allows 4-byte read, but no write */
	__u32 msg_src_ip4;	/* Allows 1,2,4-byte read and 4-byte write.
				 * Stored in network byte order.
				 */
	__u32 msg_src_ip6[4];	/* Allows 1,2,4,8-byte read and 4,8-byte write.
				 * Stored in network byte order.
				 */
	__bpf_md_ptr(struct bpf_sock *, sk);
};
```
* struct bpf_sock_ops
```
struct bpf_sock_ops {
	__u32 op;
	union {
		__u32 args[4];		/* Optionally passed to bpf program */
		__u32 reply;		/* Returned by bpf program	    */
		__u32 replylong[4];	/* Optionally returned by bpf prog  */
	};
	__u32 family;
	__u32 remote_ip4;	/* Stored in network byte order */
	__u32 local_ip4;	/* Stored in network byte order */
	__u32 remote_ip6[4];	/* Stored in network byte order */
	__u32 local_ip6[4];	/* Stored in network byte order */
	__u32 remote_port;	/* Stored in network byte order */
	__u32 local_port;	/* stored in host byte order */
	__u32 is_fullsock;	/* Some TCP fields are only valid if
				 * there is a full socket. If not, the
				 * fields read as zero.
				 */
	__u32 snd_cwnd;
	__u32 srtt_us;		/* Averaged RTT << 3 in usecs */
	__u32 bpf_sock_ops_cb_flags; /* flags defined in uapi/linux/tcp.h */
	__u32 state;
	__u32 rtt_min;
	__u32 snd_ssthresh;
	__u32 rcv_nxt;
	__u32 snd_nxt;
	__u32 snd_una;
	__u32 mss_cache;
	__u32 ecn_flags;
	__u32 rate_delivered;
	__u32 rate_interval_us;
	__u32 packets_out;
	__u32 retrans_out;
	__u32 total_retrans;
	__u32 segs_in;
	__u32 data_segs_in;
	__u32 segs_out;
	__u32 data_segs_out;
	__u32 lost_out;
	__u32 sacked_out;
	__u32 sk_txhash;
	__u64 bytes_received;
	__u64 bytes_acked;
	__bpf_md_ptr(struct bpf_sock *, sk);
};
```
* struct bpf_fib_lookup

* #define __BPF_FUNC_MAPPER(FN)
* struct __sk_buff          // user accessible mirror of in-kernel sk_buff
```
struct __sk_buff {
	__u32 len;
	__u32 pkt_type;
	__u32 mark;
	__u32 queue_mapping;
	__u32 protocol;
	__u32 vlan_present;
	__u32 vlan_tci;
	__u32 vlan_proto;
	__u32 priority;
	__u32 ingress_ifindex;
	__u32 ifindex;
	__u32 tc_index;
	__u32 cb[5];
	__u32 hash;
	__u32 tc_classid;
	__u32 data;
	__u32 data_end;
	__u32 napi_id;
	/* Accessed by BPF_PROG_TYPE_sk_skb types from here to ... */
	__u32 family;
	__u32 remote_ip4;	/* Stored in network byte order */
	__u32 local_ip4;	/* Stored in network byte order */
	__u32 remote_ip6[4];	/* Stored in network byte order */
	__u32 local_ip6[4];	/* Stored in network byte order */
	__u32 remote_port;	/* Stored in network byte order */
	__u32 local_port;	/* stored in host byte order */
	/* ... here. */

	__u32 data_meta;
	__bpf_md_ptr(struct bpf_flow_keys *, flow_keys);
	__u64 tstamp;
	__u32 wire_len;
	__u32 gso_segs;
	__bpf_md_ptr(struct bpf_sock *, sk);
	__u32 gso_size;
};
```
* struct bpf_sock
```
struct bpf_sock {
	__u32 bound_dev_if;
	__u32 family;
	__u32 type;
	__u32 protocol;
	__u32 mark;
	__u32 priority;
	/* IP address also allows 1 and 2 bytes access */
	__u32 src_ip4;
	__u32 src_ip6[4];
	__u32 src_port;		/* host byte order */
	__u32 dst_port;		/* network byte order */
	__u32 dst_ip4;
	__u32 dst_ip6[4];
	__u32 state;
	__s32 rx_queue_mapping;
};
```
* struct bpf_tcp_sock
```
struct bpf_tcp_sock {
	__u32 snd_cwnd;		/* Sending congestion window		*/
	__u32 srtt_us;		/* smoothed round trip time << 3 in usecs */
	__u32 rtt_min;
	__u32 snd_ssthresh;	/* Slow start size threshold		*/
	__u32 rcv_nxt;		/* What we want to receive next		*/
	__u32 snd_nxt;		/* Next sequence we send		*/
	__u32 snd_una;		/* First byte we want an ack for	*/
	__u32 mss_cache;	/* Cached effective mss, not including SACKS */
	__u32 ecn_flags;	/* ECN status bits.			*/
	__u32 rate_delivered;	/* saved rate sample: packets delivered */
	__u32 rate_interval_us;	/* saved rate sample: time elapsed */
	__u32 packets_out;	/* Packets which are "in flight"	*/
	__u32 retrans_out;	/* Retransmitted packets out		*/
	__u32 total_retrans;	/* Total retransmits for entire connection */
	__u32 segs_in;		/* RFC4898 tcpEStatsPerfSegsIn total number of segments in */
	__u32 data_segs_in;	/* RFC4898 tcpEStatsPerfDataSegsIn total number of data segments in. */
	__u32 segs_out;		/* RFC4898 tcpEStatsPerfSegsOut The total number of segments sent. */
	__u32 data_segs_out;	/* RFC4898 tcpEStatsPerfDataSegsOut total number of data segments sent. */
	__u32 lost_out;		/* Lost packets			*/
	__u32 sacked_out;	/* SACK'd packets			*/
	__u64 bytes_received;
	__u64 bytes_acked;
	__u32 dsack_dups;
	__u32 delivered;	/* Total data packets delivered incl. rexmits */
	__u32 delivered_ce;	/* Like the above but only ECE marked packets */
	__u32 icsk_retransmits;	/* Number of unrecovered [RTO] timeouts */
};
```

```
struct sk_psock_progs {
  struct bpf_prog			*msg_parser;
  struct bpf_prog			*skb_parser;
  struct bpf_prog			*skb_verdict;
 };
```

* struct bpf_sock_tuple

* enum xdp_action
* struct xdp_md
```
struct xdp_md {
	__u32 data;
	__u32 data_end;
	__u32 data_meta;
	/* Below access go through struct xdp_rxq_info */
	__u32 ingress_ifindex; /* rxq->dev->ifindex */
	__u32 rx_queue_index;  /* rxq->queue_index  */

	__u32 egress_ifindex;  /* txq->dev->ifindex */
};
```

* struct sk_msg_md
```
struct sk_msg_md {
	__bpf_md_ptr(void *, data);
	__bpf_md_ptr(void *, data_end);

	__u32 family;
	__u32 remote_ip4;	/* Stored in network byte order */
	__u32 local_ip4;	/* Stored in network byte order */
	__u32 remote_ip6[4];	/* Stored in network byte order */
	__u32 local_ip6[4];	/* Stored in network byte order */
	__u32 remote_port;	/* Stored in network byte order */
	__u32 local_port;	/* stored in host byte order */
	__u32 size;		/* Total size of sk_msg */

	__bpf_md_ptr(struct bpf_sock *, sk); /* current socket */
};
```
* struct sk_reuseport_md

### arch/arm64/net/bpf_jit.h



### kernel/bpf/helpers.c ###
> If kernel subsystem is allowing eBPF programs to call this function, inside its own verifier_ops->get_func_proto() callback it should return bpf_map_lookup_elem_proto, so that verifier can properly check the arguments

The BPF helper functions for kernel. Their definitions and their prototypes. E.g.,
```
BPF_CALL_4(bpf_map_update_elem, struct bpf_map *, map, void *, key,
	   void *, value, u64, flags)
{
	WARN_ON_ONCE(!rcu_read_lock_held());
	return map->ops->map_update_elem(map, key, value, flags);
}

const struct bpf_func_proto bpf_map_update_elem_proto = {
	.func		= bpf_map_update_elem,
	.gpl_only	= false,
	.pkt_access	= true,
	.ret_type	= RET_INTEGER,
	.arg1_type	= ARG_CONST_MAP_PTR,
	.arg2_type	= ARG_PTR_TO_MAP_KEY,
	.arg3_type	= ARG_PTR_TO_MAP_VALUE,
	.arg4_type	= ARG_ANYTHING,
};
```

### kernel/bpf/syscall.c ###


### /kernel/bpf/disasm.c ###
* __BPF_FUNC_MAPPER

### include/trace/bpf_probe.h ###

### include/linux/bpf_trace.h --> trace/events/xdp.h ###
### tools/testing/selftests/bpf/bpf_trace_helper.h ###

### kernel/trace/bpf_trace.c ###
