## BPF-loader.c

```
do_load_bpf_file
  elf_begin // get struct Elf
  for each Shdr
    get_sec
      gelf_getshdr
      elf_strptr
      elf_getdata
    for map section set fd to -1
    for symbol Elf_Data symbol to the section data    
    load_elf_maps_section ==> load_maps
      bpf_create_map_node  or  bpf_create_map_in_map_node
    parse_relo_and_apply    /* process all relo sections, and rewrite bpf insns for maps */
      gelf_getrel           /* populate GElf_Rela */
      insn_idx = rel.r_offset / sizeof(struct bpf_insn)
      gelf_getsym           /* populate GElf_Sym */
      insn[insn_idx].imm = maps[map_idx].fd /* maps[map_idx].elf_offset == sym.st_value */
    load_and_attach         /* load programs */
      fd = bpf_load_program

      if socket bpf is_kprobe ==> populate_prog_array ==> sys_bpf(BPF_MAP_UPDATE_ELEM...)
      if is_raw_tracepoint ==> bpf_raw_tracepoint_open ==> sys_bpf(BPF_RAW_TRACEPOINT_OPEN,...)
      if sys_XXX ==> open(DEBUGFS "kprobe_events"; write()  //add it to kprobe the https://www.kernel.org/doc/Documentation/trace/kprobetrace.txt
      get the event fd from DEBUGFS/events/*/ID
      efd = sys_bpf(BPF_RAW_TRACEPOINT_OPEN, event_fd, ...)
      ioctl(efd, PERF_EVENT_IOC_ENABLE, 0)
      ioctl(efd, PERF_EVENT_IOC_SET_BPF, fd)

```

```
load_elf_maps_section
  elf_getscn
  elf_getdata
  gelf_getsym     // get the symbol for each map
  qsort(sym, ..., cmp_symbols)    /* Align to map_fd[] order, via sort on offset in sym.st_value */
  each map
    elf_strptr ==> set map's name
    offset = sym[i].st_value
    (struct bpf_load_map_def *)(data_maps->d_buf + offset)


```
