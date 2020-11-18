## BPF-loader.c

```
do_load_bpf_file
  elf_begin // get struct Elf
  for each Shdr
    get_sec
      elf_strptr
      elf_getdata
    for map section set fd to -1
    for symbol Elf_Data symbol to the section data    
    load_maps
      bpf_create_map_node  or  bpf_create_map_in_map_node
    parse_relo_and_apply    /* process all relo sections, and rewrite bpf insns for maps */
      gelf_getrel           /* populate GElf_Rela */
      insn_idx = rel.r_offset / sizeof(struct bpf_insn)
      gelf_getsym           /* populate GElf_Sym */
      insn[insn_idx].imm = maps[map_idx].fd /* maps[map_idx].elf_offset == sym.st_value */
    load_and_attach         /* load programs */
      bpf_load_program

      if socket bpf ==> populate_prog_array

      
```
