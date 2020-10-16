## BPF cgroup

### kernel/bpf/cgroup.c

```
__cgroup_bpf_run_filter_setsockopt
  BPF_PROG_RUN_ARRAY(cgrp->bpf.effective[BPF_CGROUP_SETSOCKOPT], &ctx, BPF_PROG_RUN);
```
