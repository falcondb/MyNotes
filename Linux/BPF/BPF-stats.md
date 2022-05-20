## Patches

### [Linux patch: bpf: cgroup hierarchical stats collection](https://lore.kernel.org/bpf/20220510001807.4132027-1-yosryahmed@google.com/)


### [bpf: enable program stats](https://www.spinics.net/lists/netdev/msg553793.html)
- commit id: 492ecee892c2a4ba6a14903d5d586ff750b7e805
- on github.com: https://github.com/torvalds/linux/commit/492ecee892c2a4ba6a14903d5d586ff750b7e805
- merged to: v5.1-rc1 Mar 17, 2019

### [sharing bpf runtime stats with /dev/bpf_stats](https://lore.kernel.org/linux-fsdevel/20200314155703.bmtojqeofzxbqqhu@ast-mbp/t/#ma9139d4c2bf0343ea13a03b5ee14abc00514aa4f)


## Others
### [Video: Adding features to perf using BPF](https://www.youtube.com/watch?v=AqYs97kIAKQ)

## Code analysis

#### kernel/bpf/syscall.c
```
bpf_enable_runtime_stats

```



bpf(BPF_PROG_GET_NEXT_ID, {start_id=225, next_id=0, open_flags=0}, 112) = 0
bpf(BPF_PROG_GET_FD_BY_ID, {prog_id=226, next_id=0, open_flags=0}, 112) = 3
bpf(BPF_OBJ_GET_INFO_BY_FD, {info={bpf_fd=3, info_len=208, info=0x7fff592f3740}}, 112) = 0
openat(AT_FDCWD, "/proc/self/fdinfo/3", O_RDONLY) = 4
