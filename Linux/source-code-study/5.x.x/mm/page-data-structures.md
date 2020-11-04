## page management

#### linux/mm_types.h
```
struct mm_struct {
	struct {
		struct vm_area_struct *mmap;		/* list of VMAs */
		struct rb_root mm_rb;
		u64 vmacache_seqnum;			/* per-thread vmacache */
		unsigned long (*get_unmapped_area) ();
		unsigned long mmap_base;	/* base of mmap area */
		unsigned long mmap_legacy_base;	/* base of mmap area in bottom-up allocations */
		unsigned long task_size;	/* size of task vm space */
		unsigned long highest_vm_end;	/* highest vma end address */
		pgd_t * pgd;
		atomic_t mm_users;
		atomic_t mm_count;
		atomic_long_t pgtables_bytes;	/* PTE page table pages */
		int map_count;			/* number of VMAs */
		spinlock_t page_table_lock;
		struct rw_semaphore mmap_sem;

		struct list_head mmlist;
		unsigned long hiwater_rss; /* High-watermark of RSS usage */
		unsigned long hiwater_vm;  /* High-water virtual memory usage */
		unsigned long total_vm;	   /* Total pages mapped */
		unsigned long locked_vm;   /* Pages that have PG_mlocked set */
		atomic64_t    pinned_vm;   /* Refcount permanently increased */
		unsigned long data_vm;	   /* VM_WRITE & ~VM_SHARED & ~VM_STACK */
		unsigned long exec_vm;	   /* VM_EXEC & ~VM_WRITE & ~VM_STACK */
		unsigned long stack_vm;	   /* VM_STACK */
		unsigned long def_flags;

		spinlock_t arg_lock; /* protect the below fields */
		unsigned long start_code, end_code, start_data, end_data;
		unsigned long start_brk, brk, start_stack;
		unsigned long arg_start, arg_end, env_start, env_end;
		unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

		struct mm_rss_stat rss_stat;
		struct linux_binfmt *binfmt;
		mm_context_t context;
		unsigned long flags; /* Must use atomic bitops to access */
		struct core_state *core_state; /* coredumping support */
		struct user_namespace *user_ns;

		struct file __rcu *exe_file;
		struct work_struct async_put_work;
	} __randomize_layout;
	unsigned long cpu_bitmap[];
};


struct vm_area_struct {
	unsigned long vm_start;
	unsigned long vm_end;
	struct vm_area_struct *vm_next, *vm_prev;
	struct rb_node vm_rb;
	unsigned long rb_subtree_gap;

	struct mm_struct *vm_mm;	/* The address space we belong to. */
	pgprot_t vm_page_prot;		/* Access permissions of this VMA. */
	unsigned long vm_flags;		/* Flags, see mm.h. */
	struct {
		struct rb_node rb;
		unsigned long rb_subtree_last;
	} shared;

	struct list_head anon_vma_chain;
	struct anon_vma *anon_vma;
	const struct vm_operations_struct *vm_ops;
	unsigned long vm_pgoff;
	struct file * vm_file;		/* File we map to (can be NULL). */
	void * vm_private_data;		/* was vm_pte (shared mem) */
	struct vm_region *vm_region;	/* NOMMU mapping region */
} __randomize_layout;

```
```
struct page {
	unsigned long flags;		/* Atomic flags, some possibly
	union {
		struct {
			struct list_head lru;
			struct address_space *mapping;
			pgoff_t index;
			unsigned long private;
		};
		struct {	/* page_pool used by netstack */
			dma_addr_t dma_addr;
		};
		struct {	/* slab, slob and slub */
			union {
				struct list_head slab_list;
				struct {	/* Partial pages */
					struct page *next;
					int pages;	/* Nr of pages left */
					int pobjects;	/* Approximate count */
				};
			};
			struct kmem_cache *slab_cache; /* not slob */
			void *freelist;		/* first free object */
			union {
				void *s_mem;	/* slab: first object */
				unsigned long counters;		/* SLUB */
				struct {			/* SLUB */
					unsigned inuse:16;
					unsigned objects:15;
					unsigned frozen:1;
				};
			};
		};
		struct {	/* Tail pages of compound page */
			unsigned long compound_head;
			unsigned char compound_dtor;
			unsigned char compound_order;
			atomic_t compound_mapcount;
		};
		struct {	/* Second tail page of compound page */
			unsigned long _compound_pad_1;	/* compound_head */
			unsigned long _compound_pad_2;
			struct list_head deferred_list;
		};
		struct {	/* Page table pages */
			unsigned long _pt_pad_1;	/* compound_head */
			pgtable_t pmd_huge_pte; /* protected by page->ptl */
			unsigned long _pt_pad_2;	/* mapping */
			union {
				struct mm_struct *pt_mm; /* x86 pgds only */
				atomic_t pt_frag_refcount; /* powerpc */
			};
			spinlock_t ptl;
		};
		struct {	/* ZONE_DEVICE pages */
			struct dev_pagemap *pgmap;
			void *zone_device_data;
		};
		struct rcu_head rcu_head;
	};

	union {		/* This union is 4 bytes in size. */
		atomic_t _mapcount;
		unsigned int page_type;
		unsigned int active;		/* SLAB */
		int units;			/* SLOB */
	};
	atomic_t _refcount;
  struct mem_cgroup *mem_cgroup;

	void *virtual;
} _struct_page_alignment;
```

#### page_alloc.c
```
/* Never use with __GFP_HIGHMEM because the returned
 * address cannot represent highmem pages. Use alloc_pages and then kmap if
 * you need to access high mem.
 */
__get_free_pages
  alloc_pages
    #define NUMA
      alloc_pages_current
    #else  
      alloc_pages_node

```
