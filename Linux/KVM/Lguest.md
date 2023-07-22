# Lguest

##Introduction

### [An introduction to lguest AT LWN](https://lwn.net/Articles/218766/)
> The core of lguest is the lg loadable module. At initialization time, this module allocates a chunk of memory and maps it into the kernel's address space just above the vmalloc area - at the top, in other words. A small hypervisor is loaded into this area; it's a bit of assembly code which mainly concerns itself with switching between the kernel and the virtualized guest. Switching involves playing with the page tables - what looks like virtual memory to the host kernel is physical memory to the guest - and managing register contents.

> The hypervisor will be present in the guest systems' virtual address spaces as well, the venerable segmentation mechanism is used to keep that code out of reach.

> DMA for a virtualized I/O subsystem

> `/proc/lguest` for administration

> the user-mode hypervisor client. Its job is to allocate a range of memory which will become the guest's "physical" memory; the guest's kernel image is then mapped into that memory range. The client code itself has been specially linked to sit high in the virtual address space, leaving room for the guest system below. Once that guest system is in place, the user-mode client performs its read on the control file, causing the guest to boot.

> A file on the host system can become a disk image for the guest, with the user-mode client handling the "DMA" requests to move blocks back and forth. Network devices can be set up to perform communication between guests.



### [Lguest AT Linux Documentation](http://lguest.ozlabs.org/lguest.txt)


## Code analysis
```
run_guest
  lguest_arch_run_guest
    run_guest_once
      asm (lcall  lguest_entry) /x86/lguest/header_32.S
        LHCALL_LGUEST_INIT int $LGUEST_TRAP_ENTRY
        LHCALL_NEW_PGTABLE int $LGUEST_TRAP_ENTRY
        initialize stack
        jmp lguest_init+__PAGE_OFFSET


lguest_init
   setup pv_info, pv_irq_ops, pv_cpu_ops, pv_mmu_ops
   reserve_top_address

   i386_start_kernel
    start_kernel
      boot_cpu_init
      page_address_init
      setup_command_line
      setup_per_cpu_areas
      page_alloc_init
      trap_init
      mm_init
      sched_init
      rcu_init
      init_IRQ
      softirq_init
      cgroup_init

      rest_init
        kernel_thread(kernel_init
        init_idle_bootup_task(current)


```
