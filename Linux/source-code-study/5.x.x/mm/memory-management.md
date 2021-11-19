## Memory management execution

#### mm/memory.c
```
 // remap_pfn_range - remap kernel memory to userspace
 remap_pfn_range
 	if (is_cow_mapping)
 		vma->vm_pgoff = pfn

 	vma->vm_flags |= VM_IO | VM_PFNMAP | VM_DONTEXPAND | VM_DONTDUMP;

 	pfn -= addr >> PAGE_SHIFT;
 	pgd = pgd_offset(mm, addr);
 	flush_cache_range(vma, addr, end);
 	do {
 		next = pgd_addr_end(addr, end);
 		remap_p4d_range(mm, pgd, addr, next, pfn + (addr >> PAGE_SHIFT), prot);
    // TO BE STUDIED
 	} while (pgd++, addr = next, addr != end);

```

#### get free pages

```
__get_free_pages
  alloc_pages
    # define NUMA
    alloc_pages_current
      __alloc_pages_nodemask
        get_page_from_freelist
          for zone in zonelist
            rmqueue
              if order == 0 for buddy
                rmqueue_pcplist ==> __rmqueue_pcplist // per cpu list
                  take LRU page
              if ALLOC_HARDER
                __rmqueue_smallest  MIGRATE_HIGHATOMIC // from the lowest possible order
                  get_page_from_free_area // get from the freelist of the order in buddy
                  expand
              __rmqueue
                __rmqueue_smallest

            prep_new_page

    # else
    alloc_pages_node  
```

#### mm/page_alloc.c

```
get_page_from_freelist
	for_next_zone_zonelist_nodemask
			node_reclaim(zone->zone_pgdat, gfp_mask, order)
			if (zone_watermark_ok()
		    rmqueue
          if order == 0
            rmqueue_pcplist /* Lock and remove page from the per-cpu list */
              local_irq_save
              this_cpu_ptr(zone->pageset)->pcp
              list = &pcp->lists[migratetype];
              __rmqueue_pcplist(list)
                if list_empty   // a batch of pages are allocated each time from buddy
                  rmqueue_bulk  // allocate a batch of pages
                    __rmqueue
                      __rmqueue_smallest
                      for (passin-order to MAX_ORDER)
                        area = &(zone->free_area[current_order]);
                        page = get_page_from_free_area(area, migratetype) ==> list_first_entry_or_null
                        if (!page) continue;
                        del_page_from_free_area(page, area)  ==> list_del(&page->lru)
                        expand(zone, page, order, current_order, area, migratetype);
                          set_page_guard
                          INIT_LIST_HEAD(&page->lru)
                          page->private = order
                        page->index = migratetype;
                list_first_entry(list)
              local_irq_restore
          if flag ALLOC_HARDER
            __rmqueue_smallest
            __rmqueue  
		    prep_new_page
          kernel_init_free_pages
          clear highmen for 1 << order pages


	/*
	 * It's possible on a UMA machine to get through all zones that are
	 * fragmented. If avoiding fragmentation, reset and try again.
	 */
	if (no_fallback) {
		alloc_flags &= ~ALLOC_NOFRAGMENT;
		goto retry;
	}

	return NULL;
}
```

#### mm/gup.c
```
get_user_pages_remote
  __get_user_pages_locked
  for each number of pages
    find_vma
    pages[i] = virt_to_page(start)
    get_page(pages[i]) // inc refcount
    vmas[i] = vma
    start = (start + PAGE_SIZE) & PAGE_MASK

get_user_pages
  __get_user_pages
    // look up a page descriptor from a user-virtual address
    page = follow_page_mask
      pgd_offset
      pud_offset
      pmd_offset
      follow_page_pte
        // get the pte addr
        pte_offset_map_lock

        // gets the "struct page" associated with a pte
        vm_normal_page
          // the three memory models define pfn_to_page differently
          pfn_to_page
            // flat memory
            mem_map + ((pfn) - ARCH_PFN_OFFSET)
            // SPARSEMEM_VMEMMAP
            vmemmap + (pfn) // vmemmap = __VMEMMAP_BASE_L4 =	0xffffea0000000000UL

```

```
// GUP ping the user pages, take x86 as a reference
__get_user_pages_fast
  pgdp = pgd_offset
  gup_pud_range
    pud_offset
    // check if huge page is enabled on this pud: pud_huge(); gup_huge_pud
    gup_pmd_range  
      pmd_offset
      // check if huge page is enabled: pmd_huge(); gup_huge_pmd()
      gup_pte_range
        pte_offset_map
          // x86-64
          pte_offset_kernel
            pmd_page_vaddr(*pmd) + pte_index(address);

        // for each page in the addr range    
        page = pte_page(pte)
          pfn_to_page(pte_pfn(pte))
            pfn_to_virt(pfn)
              // kernel addr is directly mapping with a defined offset
              __va((pfn) << PAGE_SHIFT)
            virt_to_page()

        // update the page struct array
        pages[*nr] = page
  // if there va is mapped to pfn, handle it here
  get_user_pages
```


#### mm/highmem.c

#### kmap

#### kmalloc_node

```
kmalloc_node
  # CONFIG_SLOB
  kmem_cache_alloc_node_trace


```

#### Slab
*`kmem_cache_alloc`
```
kmem_cache_alloc
__kmalloc ==> __do_kmalloc
  slab_alloc
    local_irq_save
    __do_cache_alloc
      objp = ____cache_alloc
        struct array_cache as = cpu_cache_get(struct kmem_cache)
        if ac->avail ac->entry
        else cache_alloc_refill
          alloc_block
            slab_get_obj
              page->freelist
      if !objp  ____cache_alloc_node // current node ran out of memory, try other nodes
        struct kmem_cache_node = get_node
        page = get_first_slab
        slab_get_obj
    local_irq_restore

slab_alloc_node
```
* `kmem_cache_create`
```
kmem_cache_create
```

* `kmem_cache_destroy`

* `kmem_cache_shrink`

* `kmem_cache_alloc`

* `kmem_cache_free`



#### Slub
