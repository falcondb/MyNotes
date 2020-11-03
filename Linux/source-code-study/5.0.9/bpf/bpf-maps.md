## BPF Maps Implementation in the kernel

### Data Structure Definitions
```
# kernel/bpf.h

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
	u32 btf_key_type_id;
	u32 btf_value_type_id;
	struct btf *btf;
	struct bpf_map_memory memory;
	char name[BPF_OBJ_NAME_LEN];
	bool unpriv_array;
	bool frozen; /* write-once; write-protected by freeze_mutex */
	atomic64_t refcnt ____cacheline_aligned;
	atomic64_t usercnt;
	struct work_struct work;
	struct mutex freeze_mutex;
	u64 writecnt; /* writable mmap cnt; protected by freeze_mutex */
};

struct bpf_map_memory {
	u32 pages;
	struct user_struct *user;
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
	int (*map_check_btf)(const struct bpf_map *map, ...);

	/* Prog poke tracking helpers. */
	int (*map_poke_track)(struct bpf_map *map, struct bpf_prog_aux *aux);
	void (*map_poke_untrack)(struct bpf_map *map, struct bpf_prog_aux *aux);
	void (*map_poke_run)(struct bpf_map *map, u32 key, struct bpf_prog *old, struct bpf_prog *new);

	/* Direct value access helpers. */
	int (*map_direct_value_addr)(const struct bpf_map *map, u64 *imm, u32 off);
	int (*map_direct_value_meta)(const struct bpf_map *map, u64 imm, u32 *off);
	int (*map_mmap)(struct bpf_map *map, struct vm_area_struct *vma);
};


```

### Key Execution Flow
```
SYSCALL_DEFINE3
  copy_from_user(attr)
  switch cmd
    map_create
      find_and_alloc_map
      if attr->map_ifindex
		    ops = &bpf_map_offload_ops
      ops->map_alloc(attr)

    map_lookup_elem
        ufd = attr->map_fd
        fdget(ufd)
        map = __bpf_map_get(f)
              f.file->private_data
        kmalloc(value_size, GFP_USER | __GFP_NOWARN)
        map->ops->map_lookup_elem
        copy_map_value
        copy_to_user to attr->value

    bpf_prog_load
      see key-execution-flow.md
    bpf_prog_attach
      see key-execution-flow.md

    bpf_raw_tracepoint_open  

```    


#### Program and Map type registration
```
# kernel/bpf/syscall.c
static const struct bpf_map_ops * const bpf_map_types[] = {
#define BPF_PROG_TYPE(_id, _name, prog_ctx_type, kern_ctx_type)
#define BPF_MAP_TYPE(_id, _ops) \
	[_id] = &_ops,
#include <linux/bpf_types.h>
};

```

### Array Map

#### Key data structures
```
const struct bpf_map_ops array_map_ops = {
	.map_alloc_check = array_map_alloc_check,
	.map_alloc = array_map_alloc,
	.map_free = array_map_free,
	.map_get_next_key = array_map_get_next_key,
	.map_lookup_elem = array_map_lookup_elem,
	.map_update_elem = array_map_update_elem,
	.map_delete_elem = array_map_delete_elem,
	.map_gen_lookup = array_map_gen_lookup,
	.map_direct_value_addr = array_map_direct_value_addr,
	.map_direct_value_meta = array_map_direct_value_meta,
	.map_mmap = array_map_mmap,
	.map_seq_show_elem = array_map_seq_show_elem,
	.map_check_btf = array_map_check_btf,
};

struct bpf_array {
	struct bpf_map map;
	u32 elem_size;
	u32 index_mask;
	struct bpf_array_aux *aux;
	union {
		char value[0] __aligned(8);
		void *ptrs[0] __aligned(8);
		void __percpu *pptrs[0] __aligned(8);
	};
};
```




#### Key execution flow
```
# kernel/bpf/arraymap.c

array_map_alloc
  calculate the expected memory size, sizeof(truct bpf_array) + max_entries * elem_size or sizeof(void *) for PERCPU
  check the memory_lock limit for the user (user->locked_vm)
  if BPF_F_MMAPABLE
    data = bpf_map_area_mmapable_alloc ==> __bpf_map_area_alloc
      if !mmapable
        area = kmalloc_node(..., GFP_USER | __GFP_NORETRY | flags, ...)

      if mmapable
        return vmalloc_user_node_flags(..., GFP_KERNEL | __GFP_RETRY_MAYFAIL | flags)

      __vmalloc_node_flags_caller
    array = data + PAGE_ALIGN(sizeof(struct bpf_array)) - offsetof(struct bpf_array, value);
  else
    bpf_map_area_alloc ==> __bpf_map_area_alloc mmapable=false

  if percpu
    bpf_array_alloc_percpu
    for array->map.max_entries
      ptr = __alloc_percpu_gfp(GFP_USER | __GFP_NOWARN)
      array->pptrs[i] = ptr


array_map_lookup_elem
  array->value + array->elem_size * (index & array->index_mask)

```
