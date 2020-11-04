## Memory

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
 		err = remap_p4d_range(mm, pgd, addr, next,
 				pfn + (addr >> PAGE_SHIFT), prot);
 		if (err)
 			break;
 	} while (pgd++, addr = next, addr != end);

```


#### linux/mmzone.h
```
struct zone {
	unsigned long _watermark[NR_WMARK];
	unsigned long watermark_boost;
	unsigned long nr_reserved_highatomic;
	long lowmem_reserve[MAX_NR_ZONES];
	int node;
	struct pglist_data	*zone_pgdat;   // node memory layout
	struct per_cpu_pageset __percpu *pageset;  // pages for per cpu
	unsigned long		zone_start_pfn;  // zone_start_paddr >> PAGE_SHIFT

	atomic_long_t		managed_pages;  // see comments in the code
	unsigned long		spanned_pages;
	unsigned long		present_pages;

	const char		*name;
	ZONE_PADDING(_pad1_) 	/* Write-intensive fields used from the page allocator */
	struct free_area	free_area[MAX_ORDER]; 	/* free areas of different sizes for buddy */

	unsigned long		flags;
	ZONE_PADDING(_pad2_) 	/* Write-intensive fields used by compaction and vmstats. */
	unsigned long percpu_drift_mark;
	bool			contiguous;
	ZONE_PADDING(_pad3_)
	atomic_long_t		vm_stat[NR_VM_ZONE_STAT_ITEMS];
	atomic_long_t		vm_numa_stat[NR_VM_NUMA_STAT_ITEMS];
} ____cacheline_internodealigned_in_smp;


/*
for buddy system
*/
struct free_area {
	struct list_head	free_list[MIGRATE_TYPES];
	unsigned long		nr_free;
};


/*
 * On NUMA machines, each NUMA node would have a pg_data_t to describe
 * it's memory layout. On UMA machines there is a single pglist_data which
 * describes the whole memory.
 */
struct pglist_data
	struct zone node_zones[MAX_NR_ZONES];
	struct zonelist node_zonelists[MAX_ZONELISTS];
	int nr_zones;
	unsigned long node_start_pfn;
	unsigned long node_present_pages; /* total number of physical pages */
	unsigned long node_spanned_pages; /* total size of physical page range, including holes */
	int node_id;

	wait_queue_head_t kswapd_wait;
	wait_queue_head_t pfmemalloc_wait;
	struct task_struct *kswapd;	/* Protected by mem_hotplug_begin/end() */
	int kswapd_order;
	enum zone_type kswapd_classzone_idx;
	int kswapd_failures;		/* Number of 'reclaimed == 0' runs */
	unsigned long		totalreserve_pages;

	spinlock_t		lru_lock;
	struct lruvec		__lruvec;
	unsigned long		flags;

	/* Per-node vmstats */
	struct per_cpu_nodestat __percpu *per_cpu_nodestats;
	atomic_long_t		vm_stat[NR_VM_NODE_STAT_ITEMS];
} pg_data_t;


struct per_cpu_pageset {
	struct per_cpu_pages pcp;
  ...
};


struct per_cpu_pages {
	int count, high, batch
	struct list_head lists[MIGRATE_PCPTYPES];
};
```


### Slab
#### slab.h
```
struct kmem_cache {
	unsigned int object_size;/* The original size of the object */
	unsigned int size;	/* The aligned/padded/added on size  */
	unsigned int align;	/* Alignment as calculated */
	slab_flags_t flags;	/* Active flags on the slab */
	unsigned int useroffset;/* Usercopy region offset */
	unsigned int usersize;	/* Usercopy region size */
	const char *name;	/* Slab name for sysfs */
	int refcount;		/* Use counter */
	void (*ctor)(void *);	/* Called on object slot creation */
	struct list_head list;	/* List of all slab caches on the system */
};
```

#### slab-def.h
```
struct kmem_cache {

}
```

#### slab.c
```
struct array_cache {
	unsigned int avail;
	unsigned int limit;
	unsigned int batchcount;
	unsigned int touched;
	void *entry[];
};
```
