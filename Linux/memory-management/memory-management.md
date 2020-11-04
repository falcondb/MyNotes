## Memory management

### Notes
* virtual to phyical by cr3 = & page directory (x86)
* Page Frame Number (PFN)
* address types
  * User virtual addresses
  * Physical addresses
  * Bus addresses
  * Kernel logical addresses. make up the normal address space of the kernel. `kmalloc` On most architectures, logical addresses and their associated physical addresses differ only by a constant offset.
  * Kernel virtual addresses. Not necessarily have the linear. `vmalloc()`, `kmap`
  * Before accessing a specific high-memory page, the kernel must set up an explicit virtual mapping to make that page available in the kernel’s address space. Thus, many kernel data structures must be placed in low memory; high memory tends to be reserved for user-space process pages.
  * _lowmem_: logical addresses exist in kernel space contiguously mapped in physical memory, up to PAGE_OFFSET, `__pa`, `__va`
  * _highmem_: arbitrarily map physical memory, beyond the address range set aside for kernel virtual addresses.
  * `kmalloc` in lowmem; `vmalloc()` in highmem

* logical addresses are not available for high memory, `struct page` is used to keep track of just about everything the kernel needs to know about physical memory. One `struct page` is each physical page on the system
  * `struct page.flags`: PG_locked, locked in memory; PG_reserved, prevents the memory management system from working with the page at all.



### Understanding the Linux Virtual Memory Manager by Mel Gorman

#### Chapter 6  Physical Page Allocation
> Binary Buddy Allocator

> An array of free_area_t structs are maintained for each order that points to a linked list of blocks of pages
```
# v5.0.9
struct free_area {
	struct list_head	free_list[MIGRATE_TYPES];
	unsigned long		nr_free;
};

struct zone {
  struct free_area	free_area[MAX_ORDER];
}
```


#### Chapter 7  Non-Contiguous Memory Allocation
> It is preferable when dealing with large amounts of memory to use physically contiguous pages. vmalloc() allocates non-contiguous physically memory.

> between _VMALLOC_START_ and _VMALLOC_END_, at least VMALLOC_RESERVE in size. Each area is separated by at least one page to protect against overruns

>non-contiguous memory is presented by
```
struct vm_struct {
	struct vm_struct	*next;
	void			*addr;  // starting address
	unsigned long		size;
	unsigned long		flags;  // VM_ALLOC with vmalloc() or VM_IOREMAP when ioremap to map high memory
	struct page		**pages;
	unsigned int		nr_pages;
	phys_addr_t		phys_addr;   // u32 or u64
	const void		*caller;
};
```

#### Chapter 8  Slab Allocator

#### Chapter 9  High Memory Management
##### Persistent Kernel Map (PKMap)
Only for 32-bits address system?


### get user pages (GUP) & pin user pages
[pin_user_pages.rst](https://www.kernel.org/doc/Documentation/core-api/pin_user_pages.rst)
>FOLL_PIN: is internal to gup to set the correct combination of the flags

>FOLL_LONGTERM: to avoid creating a large number of wrapper functions to cover
all combinations of get*(), pin*(), FOLL_LONGTERM

>use pin_user_pages*() for DMA-pinned pages, and get_user_pages*() for other cases

>FOLL_PIN and FOLL_GET are mutually exclusive for a given gup call. However,
multiple threads and call sites are free to pin the same struct pages

>FOLL_LONGTERM is a specific case, more restrictive case of FOLL_PIN.

The rest need further studies, I can't follow at this time.

### High memory
[Why highmem](https://linux-mm.org/HighMemory)
> Coping with HighMemory
>> Memory above the physical address of 896MB are temporarily mapped into kernel virtual memory whenever the kernel needs to access that memory.
>> Data which the kernel frequently needs to access is allocated in the lower 896MB of memory (ZONE_NORMAL)
>> Data which the kernel only needs to access occasionally, including page cache, process memory and page tables, are preferentially allocated from ZONE_HIGHMEM
>> The system can have additional physical memory zones to deal with devices that can only perform DMA to a limited amount of physical memory, ZONE_DMA and ZONE_DMA32
> Temporary mapping
>> The temporary mapping of data from highmem into kernel virtual memory is done using the functions kmap(). Good SMP scalability can be obtained by using kmap_atomic(), which is lockless.


### MM on LWN
[Virtual Memory I: the problem](https://lwn.net/Articles/75174/)
> On 32-bit systems, memory is now divided into "high" and "low" memory. Low memory continues to be mapped directly into the kernel's address space, and is thus always reachable via a kernel-space pointer. High memory, instead, has no direct kernel mapping. When the kernel needs to work with a page in high memory, it must explicitly set up a special page table to map it into the kernel's address space first. This operation can be expensive, and there are limits on the number of high-memory pages which can be mapped at any particular time.

>For the most part, the kernel's own data structures must live in low memory. Memory which is not permanently mapped cannot appear in linked lists (because its virtual address is transient and variable), and the performance costs of mapping and unmapping kernel memory are too high. High memory is useful for process pages and some kernel tasks (I/O buffers, for example), but the core of the kernel stays in low memory.

[A deep dive into CMA](https://lwn.net/Articles/486301/)
[An introduction to compound pages](https://lwn.net/Articles/619514/)
>A compound page is simply a grouping of two or more physically contiguous pages into a unit that can, in many ways, be treated as a single, larger page. They are most commonly used to create huge pages, used within hugetlbfs or the transparent huge pages subsystem

>Allocating a compound page is a matter of calling a normal memory allocation function like alloc_pages() with the __GFP_COMP allocation flag set and an order of at least one

>The first (normal) page in a compound page is called the "head page"; it has the PG_head flag set. All other pages are "tail pages"; they are marked with PG_tail. all pages in a compound page have the PG_compound flag set, and the tail pages have PG_reclaim set as well.

>By default, free_compound_page() is used; all it does is return the memory to the page allocator. The hugetlbfs subsystem, though, uses free_huge_page() to keep its accounting up to date.
