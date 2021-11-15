## [The Definitive KVM Linux Doc](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt)

### General description
The ioctls belong to three classes:
  - System ioctls
  - VM ioctls
  - vcpu ioctls
  - device ioctls

### File descriptors
 - `fd` by open `/dev/kvm`
 - can be migrated among processes
 - a VM's lifecycle is *associated with its file descriptor, not its creator*

### API description
  - KVM_CREATE_VM: The new VM has no virtual cpus and no memory.
  - KVM_CREATE_VCPU: This API adds a vcpu to a virtual machine.
  - KVM_SET_USER_MEMORY_REGION: This ioctl allows the user to create, modify or delete a guest physical memory slot
  - KVM_GET_DIRTY_LOG: a bitmap containing any pages dirtied since the last call to this ioctl
  - KVM_RUN: run a guest virtual cpu
  - KVM_GET_REGS / KVM_SET_REGS / KVM_GET_SREGS / KVM_SET_SREGS
  - KVM_SET_ONE_REG: a single vcpu register can be set to a specific value
  - KVM_TRANSLATE: x86 only, Translates a virtual address according to the vcpu's current address translation mode
  - KVM_INTERRUPT: x86, ppc, mips, Queues a hardware interrupt vector to be injected
  - KVM_GET_MSRS / KVM_SET_MSRS: x86
  - KVM_SET_CPUID: x86
  - KVM_SET_SIGNAL_MASK: which signals are blocked during execution of KVM_RUN
  - KVM_GET_CLOCK / KVM_SET_CLOCK
  - KVM_GET_VCPU_EVENTS / KVM_SET_VCPU_EVENTS: Gets currently pending exceptions, interrupts, and NMIs as well as related states of the vcpu
  - KVM_SET_TSS_ADDR: defines the physical address of a three-page region in the guest physical address space
  - KVM_GET_MP_STATE / KVM_SET_MP_STATE: the vcpu's current "multiprocessing state"
  - KVM_IOEVENTFD: attaches or detaches an ioeventfd to a legal pio/mmio address. A guest write in the registered address will signal the provided event *instead of triggering an exit*. Bypass the hypervisor process and avoid context switch for performance reason. If `struct kvm_ioeventfd.datamatch` flag is set, the event will be signaled only if the written value to the registered address is equal to datamatch.
  - KVM_IRQFD: Allows setting an eventfd to directly trigger a guest interrupt. kvm_irqfd.fd specifies the file descriptor to use as the eventfd and kvm_irqfd.gsi specifies the irqchip pin toggled by this event.  When an event is triggered on the eventfd, an interrupt is injected into the guest using the specified gsi pin.

### `kvm_run` structure
Definition in [source code](https://elixir.bootlin.com/linux/v4.20.17/source/include/uapi/linux/kvm.h#L248)
  - `request_interrupt_window`: Request that KVM_RUN return when it becomes possible to inject external interrupts into the guest.  Useful in conjunction with KVM_INTERRUPT.
  - `immediate_exit`: This field is polled once when KVM_RUN starts; if non-zero, KVM_RUN exits immediately, returning -EINTR.
  - `exit_reason`: why KVM_RUN has returned
  - `ready_for_interrupt_injection`: If `request_interrupt_window` has been specified, this field indicates an interrupt can be injected now with KVM_INTERRUPT
  - `if_flag`: the current interrupt flag
  - `cr8`: cr8 register. [CR8](https://en.wikipedia.org/wiki/Control_register#CR8) is a new register accessible in 64-bit mode using the REX prefix. CR8 is used to prioritize external interrupts and is referred to as the task-priority register (TPR)
  - `apic_base`: APIC BASE msr
  - union for the exit reason. See the definition for each exit reason.
  - `kvm_valid_regs`: the register classes set by the host
  - `kvm_dirty_regs`: the register classes dirtied by userspace
[Using the KVM API LWN.net](https://lwn.net/Articles/658511/)  
An example [code](https://lwn.net/Articles/658512/) of using KVM APIs to create a simple VM
Another example [code](https://github.com/dpw/kvm-hello-world)


An example of registering hypervisor virtual addresses (HVA) to guest physical addresses (GPA)
```
mem = (struct kvm_userspace_memory_region) {
		.slot			= slot_id,
		.guest_phys_addr	= guest_phys,
		.memory_size		= size,
		.userspace_addr		= userspace_addr,
	}

ioctl(kvm->vm_fd, KVM_SET_USER_MEMORY_REGION, &mem)

```

[An introduction to Clear Containers](https://lwn.net/Articles/644675/)
