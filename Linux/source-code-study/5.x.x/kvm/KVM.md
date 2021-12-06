## KVM

### Key data structures
* `/include/linux/kvm_host.h`
```
struct kvm_vcpu {
	struct kvm *kvm;
	struct list_head blocked_vcpu_list;
	struct kvm_run *run;

	int guest_xcr0_loaded;
	struct swait_queue_head wq;

	struct kvm_mmio_fragment mmio_fragments[KVM_MAX_MMIO_FRAGMENTS];
	struct {
		u32 queued;
		struct list_head queue;
		struct list_head done;
		spinlock_t lock;
	} async_pf;

	bool preempted;
	struct kvm_vcpu_arch arch;
	struct dentry *debugfs_dentry;
};
```

### Key functions
#### `kvm_main.c`

```
// called by xmx_init() and smv_init(), their module init
kvm_init
  // with the kvm_x86_ops
  kvm_arch_init

  kvm_irqfd_init

  kvm_arch_hardware_setup

  register_cpu_notifier
  register_reboot_notifier

  kmem_cache_create

  kvm_async_pf_init

  misc_register
  register_syscore_ops

  kvm_preempt_ops.sched_in = kvm_sched_in
	kvm_preempt_ops.sched_out = kvm_sched_out

  kvm_init_debug
  kvm_vfio_ops_init

```

```
kvm_dev_ioctl_create_vm
  kvm_create_vm
    kvm_arch_alloc_vm
    kvm_arch_init_vm
    hardware_enable_all

    kvm->memslots initialization
    kvm->srcu     initialization
    kvm->buses    initialization
    kvm->mm = current->mm
    kvm_eventfd_init
    locks         initialization
    kvm->devices  initialization
    kvm_init_mmu_notifier

    list_add(&kvm->vm_list, &vm_list)

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
    kvm_x86_ops->vcpu_create()  

  create_vcpu_fd
    // vcpu is the file's private data
    anon_inode_getfd("kvm-vcpu", &kvm_vcpu_fops, vcpu)

  kvm_arch_vcpu_setup
    kvm_vcpu_reset

    // see mmu.md for x86 case
    kvm_mmu_setup

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
kvm_set_memory_region
  mutex
  __kvm_set_memory_region
    if KVM_MR_CREATE
      kvm_arch_create_memslot
        slot->arch.rmap[i] = kvm_kvzalloc()
        slot->arch.lpage_info[i - 1] = kvm_kvzalloc()

    if KVM_MR_DELETE || KVM_MR_MOVE
        // make a copy of the old memslot using memcpy()
        kmemdup

        install_new_memslots
          update_memslots
            // larger comes first
            sort_memslots

          // increase the slot generation

          kvm_arch_memslots_updated
            kvm_mmu_invalidate_mmio_sptes

        kvm_iommu_unmap_pages
          // virt/kvm/iommu.c
          kvm_iommu_put_pages
            iommu_unmap __iommu_unmap
              ops = (struct iommu_domain *)domain->ops
              ops->unmap
            kvm_unpin_pages
              kvm_release_pfn_clean
                put_page

        kvm_arch_flush_shadow_memslot
          kvm_mmu_invalidate_zap_all_pages
            // Notify all vcpus to reload its shadow page table and flush TLB.
            kvm_reload_remote_mmus
              kvm_make_all_cpus_request(kvm, KVM_REQ_MMU_RELOAD)
            kvm_zap_obsolete_pages
              kvm_mmu_prepare_zap_page
              kvm_mmu_commit_zap_page
                kvm_flush_remote_tlbs
                  kvm_make_all_cpus_request(kvm, KVM_REQ_TLB_FLUSH)
                  cmpxchg(&kvm->tlbs_dirty, dirty_count, 0)

    kvm_arch_prepare_memory_region
      // for the obsolete KVM_SET_MEMORY_REGION
      memslot->userspace_addr = vm_mmap

    install_new_memslots
    // release the MR of the old
    kvm_arch_commit_memory_region(kvm, mem, &old, change)
      vm_munmap(old->userspace_addr

    if KVM_MR_CREATE) || KVM_MR_MOVE
      kvm_iommu_map_pages
        kvm_pin_pages
          gfn_to_pfn_memslot
        iommu_map
          ops->map
```

```
kvm_make_all_cpus_request
  kvm_for_each_vcpu
    kvm_make_request

    // Run a function on a set of other CPUs. /kernel/smp.c  
    smp_call_function_many smp_call_function_single
      arch_send_call_function_ipi_mask
        smp_ops.send_call_func_single_ipi
      csd_lock_wait
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
