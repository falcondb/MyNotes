## libbpf

### Key data structures

#### bpf_object
bpf_prog_load_xattr in libbpf.c populates it by reading an object file
- struct bpf_object
```
struct bpf_object {
	char name[BPF_OBJ_NAME_LEN];
	char license[64];
	__u32 kern_version;

	struct bpf_program *programs;
	size_t nr_programs;
	struct bpf_map *maps;
	size_t nr_maps;
	size_t maps_cap;
	struct bpf_secdata sections;

	bool loaded;
	bool has_pseudo_calls;
	bool relaxed_core_relocs;

	/*
	 * Information when doing elf related work. Only valid if fd
	 * is valid.
	 */
	struct {
		int fd;
		const void *obj_buf;
		size_t obj_buf_sz;
		Elf *elf;
		GElf_Ehdr ehdr;
		Elf_Data *sym bols;
		Elf_Data *data;
		Elf_Data *rodata;
		Elf_Data *bss;
		size_t strtabidx;
		struct {
			GElf_Shdr shdr;
			Elf_Data *data;
		} *reloc_sects;
		int nr_reloc_sects;
		int maps_shndx;
		int btf_maps_shndx;
		int text_shndx;
		int data_shndx;
		int rodata_shndx;
		int bss_shndx;
	} efile;
	/*
	 * All loaded bpf_object is linked in a list, which is
	 * hidden to caller. bpf_objects__<func> handlers deal with
	 * all objects.
	 */
	struct list_head list;

	struct btf *btf;
	struct btf_ext *btf_ext;

	void *priv;
	bpf_object_clear_priv_t clear_priv;

	struct bpf_capabilities caps;

	char path[];
};
```
- bpf_program
One way of reading bpf_programs is through bpf_object__for_each_program defined in libbpf.h
```
struct bpf_program {
	/* Index in elf obj file, for relocation use. */
	int idx;
	char *name;
	int prog_ifindex;
	char *section_name;
	char *pin_name;

	struct bpf_insn *insns;

	size_t insns_cnt, main_prog_cnt;
	enum bpf_prog_type type;

	struct reloc_desc {} *reloc_desc;
	int nr_reloc;
	int log_level;

	struct {
		int nr;
		int *fds;
	} instances;
	bpf_program_prep_t preprocessor;

	struct bpf_object *obj;
	void *priv;
	bpf_program_clear_priv_t clear_priv;

	enum bpf_attach_type expected_attach_type;
	__u32 attach_btf_id;
	__u32 attach_prog_fd;
	void *func_info;
	__u32 func_info_rec_size;
	__u32 func_info_cnt;

	struct bpf_capabilities *caps;

	void *line_info;
	__u32 line_info_rec_size;
	__u32 line_info_cnt;
	__u32 prog_flags;
};
```
- bpf_map
```
struct bpf_map {
	int fd;
	char *name;
	int sec_idx;
	size_t sec_offset;
	int map_ifindex;
	int inner_map_fd;
	struct bpf_map_def def;
	__u32 btf_key_type_id;
	__u32 btf_value_type_id;
	void *priv;
	bpf_map_clear_priv_t clear_priv;
	enum libbpf_map_type libbpf_type;
	char *pin_path;
	bool pinned;
	bool reused;
};

struct bpf_map_def {
	unsigned int type;
	unsigned int key_size;
	unsigned int value_size;
	unsigned int max_entries;
	unsigned int map_flags;
};

struct bpf_map_info {
	__u32 type;
	__u32 id;
	__u32 key_size;
	__u32 value_size;
	__u32 max_entries;
	__u32 map_flags;
	char  name[BPF_OBJ_NAME_LEN];
	__u32 ifindex;
	__u32 :32;
	__u64 netns_dev;
	__u64 netns_ino;
	__u32 btf_id;
	__u32 btf_key_type_id;
	__u32 btf_value_type_id;
}

#uapi/linux/bpf.h
struct xdp_md {
	__u32 data;
	__u32 data_end;
	__u32 data_meta;
	/* Below access go through struct xdp_rxq_info */
	__u32 ingress_ifindex; /* rxq->dev->ifindex */
	__u32 rx_queue_index;  /* rxq->queue_index  */
};


# tools/lib/bpf/bpf.h

struct bpf_create_map_attr {
	const char *name;
	enum bpf_map_type map_type;
	__u32 map_flags;
	__u32 key_size;
	__u32 value_size;
	__u32 max_entries;
	__u32 numa_node;
	__u32 btf_fd;
	__u32 btf_key_type_id;
	__u32 btf_value_type_id;
	__u32 map_ifindex;
	__u32 inner_map_fd;
};

struct bpf_load_program_attr {
	enum bpf_prog_type prog_type;
	enum bpf_attach_type expected_attach_type;
	const char *name;
	const struct bpf_insn *insns;
	size_t insns_cnt;
	const char *license;
	union {
		__u32 kern_version;
		__u32 attach_prog_fd;
	};
	union {
		__u32 prog_ifindex;
		__u32 attach_btf_id;
	};
	__u32 prog_btf_fd;
	__u32 func_info_rec_size;
	const void *func_info;
	__u32 func_info_cnt;
	__u32 line_info_rec_size;
	const void *line_info;
	__u32 line_info_cnt;
	__u32 log_level;
	__u32 prog_flags;
};
```

### common code

### Perf events
```
sys_perf_event_open -> syscall(__NR_perf_event_open -> kernel/events/core.c: SYSCALL_DEFINE5(perf_event_open, ...) -> list_add_tail(&event->owner_entry, &current->perf_event_list);
```

#### libbpf.c
```
bpf_object__load_xattr
  bpf_object__probe_loading // loading test for BPF_PROG_TYPE_SOCKET_FILTER
  bpf_object__probe_caps
    // testing the following calls
    bpf_object__probe_name,
    bpf_object__probe_global_data,
    bpf_object__probe_btf_func,
    bpf_object__probe_btf_func_global,
    bpf_object__probe_btf_datasec,
    bpf_object__probe_array_mmap,
    bpf_object__probe_exp_attach_type,

  bpf_object__resolve_externs
    bpf_object__read_kconfig_mem
    bpf_object__read_kconfig_file
    bpf_object__read_kallsyms_file

  bpf_object__sanitize_and_load_btf
    TOBESTUDIED

  bpf_object__sanitize_maps
    bpf_object__for_each_map
      !obj->caps.global_data ==> Error
      !obj->caps.array_mmap  ==> m->def.map_flags ^= BPF_F_MMAPABLE

	bpf_object__load_vmlinux_btf
    obj->btf_vmlinux = libbpf_find_kernel_btf()

	bpf_object__init_kern_struct_ops_maps
    for each map
      bpf_map__init_kern_struct_ops // Init the map's fields that depend on kern_btf

	bpf_object__create_maps
    if map->pin_path
      bpf_object__reuse_map
    bpf_object__create_map
      setup struct bpf_create_map_attr
      if inner_map  bpf_object__create_map(obj, map->inner_map)
      bpf_create_map_xattr
        setup union bpf_attr
        sys_bpf(BPF_MAP_CREATE, union bpf_attr)
    if bpf_map__is_internal
      bpf_object__populate_internal_map
    bpf_map__pin

	bpf_object__relocate
    if obj->btf_ext)
      bpf_object__relocate_core
    bpf_program__relocate // for .text
    bpf_program__relocate // for the others
      TOBESTUDIED

	bpf_object__load_progs
    bpf_program__load
      if !prog->preprocessor
        load_program
          setup struct bpf_load_program_attr
          bpf_load_program_xattr
            setup union bpf_attr
            sys_bpf_prog_load
              sys_bpf(BPF_PROG_LOAD, union bpf_attr)
      for each prog
          preprocessor
          load_program

  unpin any maps
  bpf_object__unload
  bpf_object__probe_caps

```
