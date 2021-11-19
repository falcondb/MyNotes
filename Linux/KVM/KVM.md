## KVM

### `kvm_main.c`
```
kvm_dev_ioctl_create_vm
  kvm_create_vm

  // ifdef KVM_COALESCED_MMIO_PAGE_OFFSET
  kvm_coalesced_mmio_init

  // inode private data is a struct kvm instance
  anon_inode_getfd("kvm-vm", &kvm_vm_fops, kvm)

```

```
kvm_vm_ioctl
  kvm = filp->private_data

  KVM_CREATE_VCPU: kvm_vm_ioctl_create_vcpu
  KVM_SET_USER_MEMORY_REGION: kvm_vm_ioctl_set_memory_region
  KVM_IRQFD: kvm_irqfd
  KVM_IOEVENTFD: kvm_ioeventfd
  KVM_SIGNAL_MSI: kvm_send_userspace_msi
  KVM_CREATE_DEVICE: kvm_ioctl_create_device

```

```
/* Almost every key step is arch-dependent */
kvm_vm_ioctl_create_vcpu
  kvm_arch_vcpu_create

  create_vcpu_fd
    // vcpu is the file's private data
    anon_inode_getfd("kvm-vcpu", &kvm_vcpu_fops, vcpu)

  kvm_arch_vcpu_setup

  kvm->vcpus[atomic_read(&kvm->online_vcpus)] = vcpu

  kvm_arch_vcpu_postcreate
```

```
struct file_operations kvm_vcpu_fops = {
	.release        = kvm_vcpu_release,
	.unlocked_ioctl = kvm_vcpu_ioctl,
	.compat_ioctl   = kvm_vcpu_compat_ioctl,
	.mmap           = kvm_vcpu_mmap,
	.llseek		= noop_llseek,
};

struct vm_operations_struct kvm_vcpu_vm_ops = {	.fault = kvm_vcpu_fault  }  
kvm_vcpu_mmap
  vma->vm_ops = &kvm_vcpu_vm_ops


kvm_vcpu_fault
  if vmf->pgoff == 0
    page = virt_to_page(vcpu->run)
  if vmf->pgoff == KVM_PIO_PAGE_OFFSET
		page = virt_to_page(vcpu->arch.pio_data)
  if vmf->pgoff == KVM_COALESCED_MMIO_PAGE_OFFSET
		page = virt_to_page(vcpu->kvm->coalesced_mmio_ring)  
  else  
    kvm_arch_vcpu_fault

    // struct vm_fault *vmf is returned back
	  vmf->page = page  
```

```
kvm_vcpu_kick
  wqp = kvm_arch_vcpu_wq(vcpu)
  if waitqueue_active(wqp)
		wake_up_interruptible(wqp)

  // if the target vcpu is not on current CPU
  if kvm_arch_vcpu_should_kick(vcpu)
		smp_send_reschedule(cpu)  
      apic->send_IPI_mask(cpumask_of(cpu), RESCHEDULE_VECTOR)
```

```
/*
 * The vCPU has executed a HLT instruction with in-kernel mode enabled.
 */
kvm_vcpu_block
  for
    prepare_to_wait(&vcpu->wq, &wait, TASK_INTERRUPTIBLE)

    if kvm_arch_vcpu_runnable(vcpu)
			kvm_make_request(KVM_REQ_UNHALT, vcpu)

    schedule
  /* end of for loop

  finish_wait(&vcpu->wq, &wait)  
```

```
kvm_read_guest
  for each segment mapping from GPA to HVA page (head and tail trailing)
    kvm_read_guest_page
      // figure out the userspace_addr (hypervisor) of the GFN (guest page num)
      gfn_to_hva_prot
        // search for the struct kvm_memory_slot mapping to this GFN
        gfn_to_memslot

        gfn_to_hva_memslot_prot
          // it figures out many pages in the slot, the addr is resolved in __gfn_to_hva_memslot
          __gfn_to_hva_many
            __gfn_to_hva_memslot
              slot->userspace_addr + (gfn - slot->base_gfn) * PAGE_SIZE
      kvm_read_hva
        __copy_from_user
```

```
pfn_to_page

```

```
gfn_to_pfn_memslot  / gfn_to_pfn_memslot_atomic
  __gfn_to_pfn_memslot
    // get the hva from the addr mapping in the kvm_memory_slot
    __gfn_to_hva_many
      hva_to_pfn
        // if can go with the fast path for a write falut request, done
        if hva_to_pfn_fast, return
          // GUP ping the user pages, take x86 as a reference
          __get_user_pages_fast
            // see notes in mm/memory-management.md
            pgdp = pgd_offset


        hva_to_pfn_slow
          if async
            get_user_page_nowait

          else
            kvm_get_user_page_io
              __get_user_pages

        vma = find_vma_intersection
        pfn = ((addr - vma->vm_start) >> PAGE_SHIFT) + vma->vm_pgoff

```

### x86
