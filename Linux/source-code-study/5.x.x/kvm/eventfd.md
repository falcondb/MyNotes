## eventfd

### eventfd in Linux kernel
ioeventfd: translate a PIO/MMIO memory write to an eventfd signal.
userspace can register a PIO/MMIO address with an eventfd for receiving notification when the memory has been touched.

[eventfd in qemu-ioeventfd](https://www.programmersought.com/article/83465096784/)

```
kvm_vm_ioctl
  case KVM_IOEVENTFD:
    copy_from_user
    kvm_ioeventfd
      kvm_assign_ioeventfd
      or kvm_deassign_ioeventfd
```

```
kvm_assign_ioeventfd
  struct _ioeventfd *p
  kvm_iodevice_init(&p->dev, &ioeventfd_ops)
  // Encapsulate the address information and hook function in _ioeventfd into kvm_io_range and put them in the range[] array of kvm->buses. Then kvm can query whether the page fault address is in the address range of the registered ioeventfd when processing the page fault.
  kvm_io_bus_register_dev

  list_add_tail(&p->list, &kvm->ioeventfds)
```

```
struct kvm_io_device_ops ioeventfd_ops = {
	.write      = ioeventfd_write,
	.destructor = ioeventfd_destructor,
}

```

`ioeventfd_write` is called by vmx when handling VM_EXIT page fault
```
// When the virtual machine writes a memory page fault, KVM first tries to trigger the write function in p->dev to check whether the page fault address meets the ioeventfd trigger condition
ioeventfd_write
  eventfd_signal
    ctx->count += n
    if (waitqueue_active(&ctx->wqh))
		  wake_up_locked_poll(&ctx->wqh, POLLIN)
        __wake_up_locked_key
          __wake_up_common
```

```
vmx module
__init vmx_init
   kvm_init
     kvm_arch_init
     kvm_irqfd_init
        create_singlethread_workqueue("kvm-irqfd-cleanup")
     kvm_arch_hardware_setup
     register_reboot_notifier
     kmem_cache_create_usercopy
     kvm_preempt_ops.sched_in = kvm_sched_in
	   kvm_preempt_ops.sched_out = kvm_sched_out
	   kvm_init_debug
     kvm_vfio_ops_init
```

```
kvm_irqfd
  kvm_irqfd_assign  or  kvm_irqfd_deassign

kvm_irqfd_assign
    // workqueue init
    INIT_WORK(&irqfd->inject, irqfd_inject)
    INIT_WORK(&irqfd->shutdown, irqfd_shutdown)

    // eventfd is at file.private_data
    eventfd = eventfd_ctx_fileget(f.file)


    // KVM_IRQFD_FLAG_RESAMPLE
      ...

    irqfd->wait.func = irqfd_wakeup
    irqfd->pt._qproc  = irqfd_ptable_queue_proc

    irqfd_update
      kvm_irq_map_gsi
        // update kvm->irq_routing[gsi]

    events = f.file->f_op->poll(f.file, &irqfd->pt)
      // poll will call irqfd->pt._qproc, which is set to irqfd_ptable_queue_proc
  	if (events & POLLIN)
      // put it in the global workqueue, will call irqfd_inject
  		schedule_work(&irqfd->inject)    
```

```
irqfd_wakeup
  if (flags & POLLIN)
    if (irq.type == KVM_IRQ_ROUTING_MSI)
      kvm_set_msi(&irq, kvm, KVM_USERSPACE_IRQ_SOURCE_ID, 1, false)
    else
			schedule_work(&irqfd->inject)

  if (flags & POLLHUP)
      irqfd_deactivate

```

```
irqfd_inject
  kvm_set_irq(kvm, KVM_USERSPACE_IRQ_SOURCE_ID, irqfd->gsi, 1, false)
  kvm_set_irq(kvm, KVM_USERSPACE_IRQ_SOURCE_ID, irqfd->gsi, 0, false)
    // kvm_irq_map_gsi then call kvm_kernel_irq_routing_entry.set
```

```
// kvm_x86_ops.handle_exit = vmx_handle_exit
vmx_handle_exit
  kvm_vmx_exit_handlers[exit_reason](vcpu)
    // in kvm_vmx_exit_handlers definition: [EXIT_REASON_EPT_MISCONFIG] = handle_ept_misconfig
    handle_ept_misconfig
      // read the GPA trigger PF through VMCS command, remember the paper?!
      gpa = vmcs_read64(GUEST_PHYSICAL_ADDRESS)

      // try eventfd first
      kvm_io_bus_write
        // srcu on kvm->bus[bus_idx]
        __kvm_io_bus_write
          // go through the devices on the bus, and see if the GPA is in the register address range. If it is in, call kvm_iodevice_write
          kvm_iodevice_write
            // ops->write should be set to ioeventfd_write
            dev->ops->write
```
