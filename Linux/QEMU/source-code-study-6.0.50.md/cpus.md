## souce code related to CPU Virtualization

### `struct CPUState` in `hw/core/cpu.h`
* `CPUAddressSpace *cpu_ases`
* `AddressSpace *as`
* `MemoryRegion *memory`
* `KVMState *kvm_state`
* `kvm_run *kvm_run`

### `softmmu/cpus.c`
```
qemu_init_vcpu
  // populate the CPUState from MachineState qdev_get_machine()

  // defalult address space
  cpu_address_space_init(cpu->memory)
    // softmmu/physmem.c
    // TO BE STUDIED

  AccelOpsClass* cpus_accel->create_vcpu_thread(cpu)

```


```
qemu_init_cpu_loop
  qemu_init_sigbus
    action.sa_flags = SA_SIGINFO
    action.sa_sigaction = sigbus_handler
    // Bus error
    sigaction(SIGBUS, &action, NULL)

    // Early kill means that the thread receives a SIGBUS signal as soon as
    //hardware memory corruption is detected inside its address space.
    prctl(PR_MCE_KILL, PR_MCE_KILL_SET, PR_MCE_KILL_EARLY, 0, 0)
```

```
vm_stop

  if qemu_in_vcpu_thread ==> qemu_cpu_is_self(current_cpu)
    qemu_system_vmstop_request
      qemu_notify_event

    cpu_stop_current
      cpu_exit  // hw/core/cpu.c
      qatomic_set(&cpu->exit_request, 1)

  else
    do_vm_stop
      pause_all_vcpus()
        // see its section

      vm_state_notify(0, state)
        // call the callbacks in the vm_change_state_head

      qapi_event_send_stop()

      bdrv_drain_all  // block/io.c
}
```


```
pause_all_vcpus
  CPU_FOREACH
    if qemu_cpu_is_self
      qemu_cpu_stop
        cpu_exit
        qemu_cond_broadcast(&qemu_pause_cond)
    else
      qemu_cpu_kick

  // Don't understand the rest code:
  // drop the replay_lock so any vCPU threads woken up can finish their replay tasks
```

```
qemu_cpu_kick
  qemu_cond_broadcast(cpu->halt_cond)
    // with accel
    cpus_accel->kick_vcpu_thread(cpu)
    // or
    cpus_kick_thread
      pthread_kill(cpu->thread->thread, SIG_IPI)
```


# `target/i386/cpu.c`

```
type_init(x86_cpu_register_types)

x86_cpu_register_types
   type_register_static(&x86_cpu_type_info)

   x86_register_cpudef_types(&builtin_x86_defs[i]);

   type_register_static(&max_x86_cpu_type_info);
   type_register_static(&x86_base_cpu_type_info);
   type_register_static(&host_x86_cpu_type_info);
```


```
TypeInfo x86_cpu_type_info = {
    .name = TYPE_X86_CPU,
    .parent = TYPE_CPU,
    .instance_size = sizeof(X86CPU),
    .instance_init = x86_cpu_initfn,
    .abstract = true,
    .class_size = sizeof(X86CPUClass),
    .class_init = x86_cpu_common_class_init,
};


TypeInfo max_x86_cpu_type_info = {
    .name = X86_CPU_TYPE_NAME("max"),
    .parent = TYPE_X86_CPU,
    .instance_init = max_x86_cpu_initfn,
    .class_init = max_x86_cpu_class_init,
};


static const TypeInfo x86_base_cpu_type_info = {
        .name = X86_CPU_TYPE_NAME("base"),
        .parent = TYPE_X86_CPU,
        .class_init = x86_cpu_base_class_init,
};


TypeInfo host_x86_cpu_type_info = {
    .name = X86_CPU_TYPE_NAME("host"),
    .parent = X86_CPU_TYPE_NAME("max"),
    .class_init = host_x86_cpu_class_init,
};

```

```
x86_cpu_common_class_init
  device_class_set_parent_realize(x86_cpu_realizefn, &xcc->parent_realize)

x86_cpu_realizefn

  x86_cpu_expand_features
  x86_cpu_filter_features
    // TO BE STUDIED

  cpu_exec_realizefn
    // TO BE STUDIED

  mce_init

  /* HERE WE GO */
  qemu_init_vcpu


  x86_cpu_apic_realize

  cpu_reset

  xcc->parent_realize()

```
