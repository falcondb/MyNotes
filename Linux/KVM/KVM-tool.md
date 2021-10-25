## Native Linux KVM tool

### On LWN.net
[Native Linux KVM tool](https://lwn.net/Articles/436781/)

### Source code
At `tools/kvm/`

### Key Executions

#### setup
```
kvm_cmd_setup ==> do_setup
  make_dir(guestfs_name) // make the root fs for the guest system
kvm->cfg
  copy_init     // Copy the root image to virt/init of the guest system as an initramfs

  copy_passwd   // set root user in /etc/passwd of the guest system
```

#### run
```
kvm_cmd_run
  kvm_cmd_run_init
    // read cmd options and set up kvm->cfg
    // call the struct init_item.init() registered in init_lists. see util/init.c util-init.h
  kvm_cmd_run_work
    foreach kvm->nrcpus
      pthread_create(&kvm->cpus[i]->thread, NULL, kvm_cpu_thread, kvm->cpus[i])
  kvm_cmd_run_exit
    init_list__exit
```

```
kvm_cpu_thread
  kvm_cpu__start
    // block SIGALAM
    signal(SIGKVMEXIT, kvm_cpu_signal_handler)
    signal(SIGKVMPAUSE, kvm_cpu_signal_handler)
    kvm_cpu__reset_vcpu  // reset virtual CPU to a known state

    while cpu->is_running


      kvm_cpu__run  ==>  ioctl(vcpu->vcpu_fd, KVM_RUN, 0)

      switch cpu->kvm_run->exit_reason
        case KVM_EXIT_IO:
          kvm_cpu__emulate_io

        case KVM_EXIT_MMIO:
          kvm_cpu__handle_coalesced_mmio
          kvm_cpu__emulate_mmio


```

```
kvm/ioport.c
kvm_cpu__emulate_io   // arch dependent
  kvm__emulate_io   ==>   kvm__emulate_io

```
