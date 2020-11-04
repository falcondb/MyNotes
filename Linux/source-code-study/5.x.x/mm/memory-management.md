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

#### Slab

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

#### Slub
