## BPF cgroup

### kernel/bpf/cgroup.c

```
__cgroup_bpf_run_filter_setsockopt
  BPF_PROG_RUN_ARRAY(cgrp->bpf.effective[BPF_CGROUP_SETSOCKOPT], &ctx, BPF_PROG_RUN);
```

## cgroup_helper
linux/tools/testing/selftests/bpf/cgroup_helpers.c

```
setup_cgroup_environment
  unshare(CLONE_NEWNS)
  mount("none", "/", ...) // mount fakeroot! why?
  mount("none", CGROUP_MOUNT_PATH, "cgroup2", 0, NULL)
  mkdir(cgroup_workdir, 0777)
  enable_all_controllers // cgroup V2
    open and read cgroup.controllers and cgroup.subtree_control  
```
