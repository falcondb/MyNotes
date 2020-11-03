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
  __gup_longterm_locked
  kcalloc(nr_pages, sizeof(struct vm_area_struct *), GFP_KERNEL)
    __get_user_pages_locked
    check_and_migrate_cma_pages
```


#### mm/highmem.c

#### kmap

#### kmalloc_node

```
kmalloc_node
  # CONFIG_SLOB
  kmem_cache_alloc_node_trace


```
