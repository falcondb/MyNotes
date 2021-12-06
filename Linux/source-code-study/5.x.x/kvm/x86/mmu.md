### MMU
*Better to go through the [slides](https://www.linux-kvm.org/images/e/e5/KvmForum2007%24shadowy-depths-of-the-kvm-mmu.pdf) before reading the complicated details in the source code*

#### Key data structures
* `/arch/x86/include/asm/kvm_host.h`
Refer to my Linux/KVM/MMU.md and [The x86 kvm shadow mmu](https://www.kernel.org/doc/Documentation/virtual/kvm/mmu.txt)
```
struct kvm_mmu_page {
	struct list_head link;
	struct hlist_node hash_link;
	gfn_t gfn;
	union kvm_mmu_page_role role;

  //
	u64 *spt;
	/* hold the gfn of each spte inside spt */
	gfn_t *gfns;
  ...
}
```

```
struct kvm_mmu_page {
	struct list_head link;
	struct hlist_node hash_link;

	union kvm_mmu_page_role role;
	gfn_t gfn;

	u64 *spt;
	/* hold the gfn of each spte inside spt */
	gfn_t *gfns;
	struct kvm_rmap_head parent_ptes; /* rmap pointers to parent sptes */
  ...
};
```

#### Init
```
kvm_arch_vcpu_setup
  kvm_mmu_setup
    init_kvm_mmu
      // TDP, EPT(Intel), NPT(AMD), SLAT: hardware-assisted virtualization
      init_kvm_tdp_mmu
        struct kvm_mmu *context = vcpu->arch.walk_mmu;
        context->page_fault = tdp_page_fault;
        context->sync_page = nonpaging_sync_page;
        context->invlpg = nonpaging_invlpg;
        context->update_pte = nonpaging_update_pte;
        context->shadow_root_level = kvm_x86_ops->get_tdp_level();
        context->root_hpa = INVALID_PAGE;
        context->direct_map = true;
        context->set_cr3 = kvm_x86_ops->set_tdp_cr3;
        context->get_cr3 = get_cr3;
        context->get_pdptr = kvm_pdptr_read;
        context->inject_page_fault = kvm_inject_page_fault;

      // or
      init_kvm_softmmu  / nested_svm_init_mmu_context
        vcpu->arch.walk_mmu->set_cr3           = kvm_x86_ops->set_cr3;
        vcpu->arch.walk_mmu->get_cr3           = get_cr3;
        vcpu->arch.walk_mmu->get_pdptr         = kvm_pdptr_read;
        vcpu->arch.walk_mmu->inject_page_fault = kvm_inject_page_fault;

        kvm_init_shadow_mmu
          nonpaging_init_context

          //or
          paging64_init_context
            paging64_init_context_common
              context->page_fault = paging64_page_fault;
              context->gva_to_gpa = paging64_gva_to_gpa;
              context->sync_page = paging64_sync_page;
              context->invlpg = paging64_invlpg;
              context->update_pte = paging64_update_pte;
              context->shadow_root_level = level;
              context->root_hpa = INVALID_PAGE;
              context->direct_map = false;

          //or
          paging32E_init_context

          //or
          paging32_init_context

```

```
// called by vcpu_enter_guest, then kvm_mmu_reload
kvm_mmu_load
  mmu_topup_memory_caches
  // or
  mmu_alloc_roots
  kvm_mmu_sync_roots
  // vmx_set_cr3 for vmx
  vcpu->arch.mmu.set_cr3(vcpu, vcpu->arch.mmu.root_hpa)

```

```
kvm_mmu_page_fault
  vcpu->arch.mmu.page_fault

  x86_emulate_instruction

```

- the page fault handlers for guest 32 and guest 64 are defined in `paging_tmpl.h`

```
tdp_page_fault
  mmu_topup_memory_caches

  // try fast path and done
  fast_page_fault

  // try async pf and done
  try_async_pf

  make_mmu_pages_available

  transparent_hugepage_adjust

  __direct_map
```

```
nonpaging_page_fault
	mmu_topup_memory_caches
	nonpaging_map
		// only if the shadow page table is present and it is caused by write-protect
		if fast_page_fault() done

		// TO BE STUDIED: async_pf()
		if try_async_pf() done

		make_mmu_pages_available
		// update the SPT with the fault gfn
		__direct_map

```


```
// SLAB / SLUB / SLOB for these struct kmem_cache objects and cached pages
mmu_topup_memory_caches
  mmu_topup_memory_cache(&vcpu->arch.mmu_pte_list_desc_cache, pte_list_desc_cache, 8 + PTE_PREFETCH_NUM)
  mmu_topup_memory_cache_page(&vcpu->arch.mmu_page_cache, 8)
  mmu_topup_memory_cache(&vcpu->arch.mmu_page_header_cache, mmu_page_header_cache, 4)
```

```
make_mmu_pages_available
	// if avaibable pages > KVM_MIN_FREE_MMU_PAGES, recycling is not needed. done
	while (kvm_mmu_available_pages(vcpu->kvm) < KVM_REFILL_PAGES)
		if (!prepare_zap_oldest_mmu_page(vcpu->kvm, &invalid_list))
			break
	kvm_mmu_commit_zap_page(vcpu->kvm, &invalid_list)

```

```
__direct_map
  for_each_shadow_entry
    if iterator.level == level
			// at the target level, update the entry in this SPT with the gfn
      mmu_set_spte
      direct_pte_prefetch
      break

    drop_large_spte
    if !is_shadow_present_pte()
      kvm_mmu_get_page
				// see kvm_mmu_get_page section

      link_shadow_page
				spte = __pa(sp->spt) | XXXX
				mmu_spte_set(sptep, spte)
```

```
mmu_set_spte
  if is_rmap_spte()			// is_shadow_present_pte and !is_mmio_spte
	  if pfn != spte_to_pfn(*sptep)
	    drop_spte
	      if mmu_spte_clear_track_bits
		       rmap_remove
	          gfn = kvm_mmu_page_get_gfn
	          rmapp = gfn_to_rmap
	            slot = gfn_to_memslot
	            __gfn_to_rmap
	              idx = gfn_to_index(gfn, slot->base_gfn, level)
	              return &slot->arch.rmap[level - PT_PAGE_TABLE_LEVEL][idx]
	          pte_list_remove(spte, rmapp)
	    kvm_flush_remote_tlbs

	if set_spte()
				kvm_make_request(KVM_REQ_TLB_FLUSH, vcpu)
```

```
set_spte
	spte = PT_PRESENT_MASK
	// set different masks on spte
	...
	spte |= (u64)pfn << PAGE_SHIFT

	if (pte_access & ACC_WRITE_MASK)
		mark_page_dirty

	if mmu_spte_update()
		kvm_flush_remote_tlbs
```

```
mmu_spte_update
	if (!is_shadow_present_pte(old_spte)) {
		mmu_spte_set(sptep, new_spte)
			// for x86-64, just set the new 64bits, otherwise, set the two 32bits
			__set_spte
		return

	// other cases to update spte state
```

```
kvm_mmu_get_page
	for_each_gfn_sp
		mmu_page_add_parent_pte

		// handle unsynced children and parent itself
		return

	// OK, no match sp with the gfn
	kvm_mmu_alloc_page

	hlist_add_head(&sp->hash_link, )

```

```
kvm_mmu_prepare_zap_page
	mmu_zap_unsync_children
		kvm_mmu_pages_init
		while (mmu_unsync_walk(parent, &pages)
			for_each_sp(pages, sp, parents, i)
				kvm_mmu_prepare_zap_page(kvm, sp, invalid_list);
				mmu_pages_clear_parents(&parents);

	kvm_mmu_pages_init(parent, &parents, &pages)
	kvm_mmu_page_unlink_children
		// for each
		mmu_page_zap_pte(kvm, sp, sp->spt + i)
	kvm_mmu_unlink_parents

```

```
mmu_unsync_walk
	// add a kvm_mmu_page to kvm_mmu_pages
	mmu_pages_add

	__mmu_unsync_walk
		for_each_set_bit(i, sp->unsync_child_bitmap, ...)

			child = page_header
			// if kvm_mmu_page child has unsynced children or itself is unsynced
			// add the children or itself to the kvm_mmu_pages

```

#### `page_tmpl.h` 32-bit and 64-bit common code

```
FNAME(page_fault)
	mmu_topup_memory_caches

	FNAME(walk_addr)

	if try_async_pf() return

	FNAME(fetch)

	for (shadow_walk_init; shadow_walk_okay() && it.level > gw->level; shadow_walk_next())
		drop_large_spte

```

```
FNAME(fetch)
	// to the fault page level
	for shadow_walk.level > gw->level
		drop_large_spte

		if !is_shadow_present_pte
			kvm_mmu_get_page

		link_shadow_page
				mmu_spte_set

	// to the PTEs for the addr
	for shadow_walk.level > hlevel
		drop_large_spte

		if is_shadow_present_pte
			continue

		kvm_mmu_get_page
		link_shadow_page

	mmu_set_spte

	FNAME(pte_prefetch)
```

For the cases of protected mode and not _TDP_ enabled, the guest address walking needs to be done in software kvm.
```
walk_addr_generic
	walker->level = mmu->root_level
	pte           = mmu->get_cr3(vcpu)
	--walker->level

	do // the entry of walking each level of the page tables (dictionaries)
		// the index of the address in current level of table
		index = PT_INDEX(addr, walker->level);
		table_gfn = gpte_to_gfn(pte);
		// pt_element_t is 32 bit or 64 bit entry for x86-32 or x86-64
		offset    = index * sizeof(pt_element_t);
		pte_gpa   = gfn_to_gpa(table_gfn) + offset;

		// handle the nested MMU case
		mmu->translate_gpa

		host_addr = gfn_to_hva_prot		// see KVM.md, host addr mapping through kvm_memory_slot

		// copy the PTE at host_addr + offset from user to the pte in walker

		walker->ptes[walker->level - 1] = pte
	while !is_last_gpte()

	// the base of the last PTE
	gfn = gpte_to_gfn_lvl(pte, walker->level);
	// the gfn of PTE entry for the addr
	gfn += (addr & PT_LVL_OFFSET_MASK(walker->level)) >> PAGE_SHIFT;

	// for nested MMU
	real_gpa = mmu->translate_gpa

	walker->gfn = real_gpa >> PAGE_SHIFT
```
