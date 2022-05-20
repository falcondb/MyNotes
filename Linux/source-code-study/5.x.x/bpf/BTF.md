## BTF
bpf_object__load_vmlinux_btf / libbpf_find_vmlinux_btf_id

  btf__load_vmlinux_btf
    btf__parse_elf


 bpf_program__set_attach_target / bpf_object_load
      bpf_object__load_vmlinux_btf


 btf_parse_raw
      // read the raw data from the file
      btf_new
          btf_parse_hdr
          btf_parse_str_sec
          btf_parse_type_sec
              btf_add_type_idx_entry
                btf_add_type_offs_mem
                  libbpf_add_mem
