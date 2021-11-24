## Memory management

### Notes
* virtual to phyical by register `cr3` = & page directory (x86)
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
  * x86-64, if `! CONFIG_DYNAMIC_MEMORY_LAYOUT` both _VMALLOC_START_ and _VMEMMAP_START_ are set to `__VMALLOC_BASE_L4 0xffffc90000000000UL`; if`CONFIG_DYNAMIC_MEMORY_LAYOUT`, _VMALLOC_START_ and _VMEMMAP_START_  are set to `vmalloc_base` dynamically.
  * x86-set, _VMALLOC_START_ is set to `high_memory + VMALLOC_OFFSET (8MB)`, `high_memory` to `-128UL << 20` in `mm/init_32.c`

### Understanding the Linux Virtual Memory Manager by Mel Gorman

#### Chapter 6  Physical Page Allocation
> Binary Buddy Allocator

> An array of free_area_t structs are maintained for each order that points to a linked list of blocks of pages

```
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
> The Contiguous Memory Allocator (or CMA) allows allocation of big, physically-contiguous memory blocks

> CMA works by reserving memory early at boot time. This memory, called a CMA area or a CMA context, is later returned to the buddy allocator so that it can be used by regular applications.



[An introduction to compound pages](https://lwn.net/Articles/619514/)
>A compound page is simply a grouping of two or more physically contiguous pages into a unit that can, in many ways, be treated as a single, larger page. They are most commonly used to create huge pages, used within hugetlbfs or the transparent huge pages subsystem

>Allocating a compound page is a matter of calling a normal memory allocation function like alloc_pages() with the __GFP_COMP__ allocation flag set and an order of at least one

>The first (normal) page in a compound page is called the "head page"; it has the PG_head flag set. All other pages are "tail pages"; they are marked with PG_tail. all pages in a compound page have the PG_compound flag set, and the tail pages have PG_reclaim set as well.

>By default, free_compound_page() is used; all it does is return the memory to the page allocator. The hugetlbfs subsystem, though, uses free_huge_page() to keep its accounting up to date.

[Five-level page tables](https://lwn.net/Articles/717293/)
> page global directory (PGD);  page upper directory (PUD); page middle directory (PMD); page table entry (PTE); PGD, PUD, PMD are 9 bit indexed entry with 8 bytes wide (2^9 *8 = 2^12 = 4KB)

>The middle levels can also have special entries indicating that they point directly to a (large) physical page rather than to a lower-level page table; that is how huge pages are implemented.

48-bit VM address space or 128 TB may not large enough in the coming future. So,
> add another level of indirection in the form of a fifth level of page tables. The new level, called the "P4D", is inserted between the PGD and the PUD. It will support 52-bit physical addresses and 57-bit virtual addresses. The new level also allows the creation of 512GB huge pages.

>Anybody wanting to play with the new mode can do so now with QEMU, which understands five-level page tables.

[Four-level page tables merged](https://lwn.net/Articles/117749/)

[Reducing page structures for huge pages](https://lwn.net/Articles/839737/)


[Memory Management](https://www.kernel.org/doc/html/latest/admin-guide/mm/index.html)
HugeTLB Pages
> Users can use the huge page support in Linux kernel by either using the mmap system call or standard SYSV shared memory system calls

> CONFIG_HUGETLBFS & CONFIG_HUGETLB_PAGE

> applications are going to use only shmat/shmget system calls or mmap with MAP_HUGETLB

> On a NUMA platform, the kernel will attempt to distribute the huge page pool over all the set of allowed nodes specified by the NUMA memory policy of the task that modifies nr_hugepages.

```
/proc/meminfo
/proc/filesystems
/proc/sys/vm/nr_hugepages
/proc/sys/vm/
/sys/devices/system/node/node*/meminfo
/sys/kernel/mm/hugepages
numactl
```
Using Huge Pages
```
mount -t hugetlbfs ...
```

[libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs)


> On a NUMA platform, the kernel will attempt to distribute the huge page pool over all the set of allowed nodes specified by the NUMA memory policy of the task that modifies nr_hugepages.


Idle Page Tracking
> CONFIG_IDLE_PAGE_TRACKING

> Only accesses to user memory pages are tracked. These are pages mapped to a process address space, page cache and buffer pages, swap cache pages. For other page types (e.g. SLAB pages) an attempt to mark a page idle is silently ignored, and hence such pages are never reported idle.

- Mark all the workload’s pages as idle by setting corresponding bits in /sys/kernel/mm/page_idle/bitmap. The pages can be found by reading /proc/pid/pagemap if the workload is represented by a process, or by filtering out alien pages using /proc/kpagecgroup in case the workload is placed in a memory cgroup.
- Wait until the workload accesses its working set.
- Read /sys/kernel/mm/page_idle/bitmap and count the number of bits set. If one wants to ignore certain types of pages, e.g. mlocked pages since they are not reclaimable, he or she can filter them out using /proc/kpageflags.

```
/sys/kernel/mm/page_idle
/sys/kernel/mm/page_idle/bitmap
```

Kernel Samepage Merging
> CONFIG_KSM

>KSM was originally developed for use with KVM (where it was known as Kernel Shared Memory), to fit more virtual machines into physical memory, by sharing the data common between them. But it can be useful to any application which generates many instances of the same data.

> The KSM daemon ksmd periodically scans those areas of user memory which have been registered with it, looking for pages of identical content which can be replaced by a single write-protected page

> KSM only merges anonymous (private) pages, never pagecache (file) pages.

```
int madvise(void *addr, size_t length, int advice);  // give advice about use of memory
 /sys/kernel/mm/ksm/
```

Memory Hotplug
> Memory Hotplug allows users to increase/decrease the amount of memory. Generally, there are two purposes.

No-MMU memory mapping support
> The kernel has limited support for memory mapping under no-MMU conditions, such as are used in uClinux environments.

* Many differences in the memory mapping behaviors with MMU and No-MMU, see the page for the details

NUMA Memory Policy
> “memory policy” determines from which node the kernel will allocate memory in a NUMA system
> System Default Policy; Task/Process Policy; VMA Policy; Shared Policy

> A NUMA memory policy consists of a “mode”, optional mode flags, and an optional set of nodes. The mode determines the behavior of the policy, the optional mode flags determine the behavior of the mode, and the optional set of nodes can be viewed as the arguments to the policy behavior.


```
numactl --show
/sys/devices/system/node/online
/proc/$PID/task/$TASK/numa_maps

set_mempolicy
get_mempolicy
mbind   //mbind() installs the policy specified by as a VMA policy for the range of the calling task’s address space
numactl
```

```
/sys/devices/system/node/nodeX/access0/targets/
/sys/devices/system/node/nodeY/access0/initiators/
/sys/devices/system/node/nodeX/memory_side_cache/
```

* /proc/pid/pagemap
> lets a userspace process find out which physical frame each virtual page is mapped to. It contains one 64-bit value for each virtual page

* /proc/kpagecount
> contains a 64-bit count of the number of times each page is mapped, indexed by PFN

* /proc/kpageflags
> contains a 64-bit set of flags for each page, indexed by PFN

* /proc/kpagecgroup
> a 64-bit inode number of the memory cgroup each page is charged to, indexed by PFN

```
echo 4 > /proc/PID/clear_refs //Clear soft-dirty bits from the task’s PTEs
```

Transparent Hugepage Support (THP)
> Performance critical computing applications dealing with large memory working sets are already running on top of libhugetlbfs and in turn hugetlbfs. Transparent HugePage Support (THP) is an alternative mean of using huge pages for the backing of virtual memory with huge pages that supports the automatic promotion and demotion of page sizes and without the shortcomings of hugetlbfs.

> With THP, the TLB miss will run faster, and a single TLB entry will be mapping a much larger amount of virtual memory in turn reducing the number of TLB misses.

> Unless THP is completely disabled, there is khugepaged daemon that scans memory and collapses sequences of basic pages into huge pages

> The THP behaviour is controlled via sysfs interface and using _madvise_ and _prctl_ system calls

```
echo always >/sys/kernel/mm/transparent_hugepage/enabled
echo madvise >/sys/kernel/mm/transparent_hugepage/enabled
echo never >/sys/kernel/mm/transparent_hugepage/enabled

echo always >/sys/kernel/mm/transparent_hugepage/defrag
echo defer >/sys/kernel/mm/transparent_hugepage/defrag
echo defer+madvise >/sys/kernel/mm/transparent_hugepage/defrag
echo madvise >/sys/kernel/mm/transparent_hugepage/defrag
echo never >/sys/kernel/mm/transparent_hugepage/defrag

echo 0 >/sys/kernel/mm/transparent_hugepage/khugepaged/defrag
echo 1 >/sys/kernel/mm/transparent_hugepage/khugepaged/defrag
/sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan
/sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs
/sys/kernel/mm/transparent_hugepage/khugepaged/alloc_sleep_millisecs
...

grep Huge /proc/meminfo  
grep AnonHugePages /proc/PID/smaps
egrep "thp|compact" /proc/vmstat
```

Transparent Hugepage Support maximizes the usefulness of free memory if compared to the reservation approach of hugetlbfs by allowing all unused memory to be used as cache or other movable (or even unmovable entities). It doesn’t require reservation to prevent hugepage allocation failures to be noticeable from userland. It allows paging and all other advanced VM features to be available on the hugepages. It requires no modifications for applications to take advantage of it.


Userfaultfd [Userfaultfd Man page](https://man7.org/linux/man-pages/man2/userfaultfd.2.html)
> Userfaults allow the implementation of on-demand paging from userland and more generally they allow userland to take control of various memory page faults

* read/POLLIN protocol to notify a userland thread of the faults happening
* various UFFDIO_* ioctls that can manage the virtual memory regions registered in the userfaultfd that allows userland to efficiently resolve the userfaults it receives via 1) or to manage the virtual memory in the background

> QEMU/KVM is using the userfaultfd syscall to implement postcopy live migration. Postcopy live migration is one form of memory externalization consisting of a virtual machine running with part or all of its memory residing on a different node in the cloud. The userfaultfd abstraction is generic enough that not a single line of KVM kernel code had to be modified in order to add postcopy live migration to QEMU.

* Read the details here about how QEMU uses Userfaultfd

[Documentation for /proc/sys/vm/](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/vm.html)

[DMA-API-HOWTO.txt](linux/Documentation/DMA-API-HOWTO.txt)


[Memory Ordering at Compile Time](https://preshing.com/20120625/memory-ordering-at-compile-time/)
The cardinal rule of memory reordering: not modify the behavior of a single-threaded program
It’s only when lock-free techniques are used – when memory is shared between threads without any kind of mutual exclusion
Compiler explicit memory barrier: `asm volatile("" ::: "memory")`
Every function containing a compiler barrier must act as a compiler barrier itself, even when the function is inlined. In fact, the majority of function calls act as compiler barriers, whether they contain their own compiler barrier or not.

[Memory Barriers Are Like Source Control Operations](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)
_Types of Memory Barrier_
  - A LoadLoad barrier effectively prevents reordering of loads performed before the barrier with loads performed after the barrier.
  - A StoreStore barrier effectively prevents reordering of stores performed before the barrier with stores performed after the barrier
  - LoadStore
     Encounters a load, he looks ahead at any stores that are coming up after that; if the stores are completely unrelated to the current load, then he’s allowed to skip ahead, do the stores first, then come back afterwards to finish up the load. if, say, there is a cache miss on the load followed by a cache hit on the store.
  - A StoreLoad barrier ensures that all stores performed before the barrier are visible to other processors, and that all loads performed after the barrier receive the latest value that is visible at the time of the barrier.

[Weak vs. Strong Memory Models](https://preshing.com/20120930/weak-vs-strong-memory-models/)
A _memory model_ tells you, for a given processor or toolchain, exactly what types of memory reordering to expect at runtime relative to a given source code listing. Keep in mind that the effects of memory reordering can only be observed when lock-free programming techniques are used.

A _hardware memory_ model tells you what kind of memory ordering to expect at runtime relative to an assembly (or machine) code listing.

_Weak Memory Models_
Any load or store operation can effectively be reordered with any other load or store operation, as long as it would never modify the behavior of a single, isolated thread. In reality, the reordering may be due to either compiler reordering of instructions, or memory reordering on the processor itself.
Must still specify the correct memory ordering constraints, if only to prevent compiler reordering.

CPU families with weak hardware ordering:
  - _ARM_, _PowerPC_, _Itanium_

_Strong Memory Models_
A strong hardware memory model is one in which every machine instruction comes implicitly with acquire and release semantics. As a result, when one CPU core performs a sequence of writes, every other CPU core sees those values change in the same order that they were written.

[Memory: the flat, the discontiguous, and the sparse](https://lwn.net/Articles/789304/)
_FLATMEM_
  - The memory map remained a flat array that was neat and efficient, but it had a major drawback: it couldn't deal well with large holes in the physical address space.

_DISCONTIGMEM_
  - For NUMA machines
  - The DISCONTIGMEM model introduced the notion of a memory node, which remains the basis of NUMA memory management. Each node carries an independent (well, mostly) memory-management subsystem with its own free-page lists, in-use page lists, least-recently-used (LRU) information, and usage statistics.
  - The node data (struct pglist_data or pg_data_t for short) contains a node-specific memory map.
  - `pfn_to_page()  page_to_pfn()`
  - A weak point: memory hotplug and hot remove

_SPARSEMEM_
  - Abstracted the memory map as a collection of sections (`struct mem_section`) of arbitrary size defined by the architectures.
  - The array of these sections is a meta-memory map which can be efficiently chopped at SECTION_SIZE granularity.
  - Several high bits of the PFN are used to index into the sections array.
  - _SPARSEMEM_EXTREME_ added a second dimension to the sections array and made this array, well, sparse.
  - [SPARSEMEM_VMEMMAP](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8f6aac419bd) (`/mm/sparse-vmemmap.c`)
  - The entire memory map is mapped into a virtually contiguous area, but only the active sections are backed with physical pages.
  - To support _persistent-memory devices_, the virtual memory map can use struct `vmem_altmap`, which will provide page structures in persistent memory
  - In 2008, _SPARSEMEM_VMEMMAP_ became *the only supported memory model for x86-64*, as it was only slightly more expensive than _FLATMEM_ but more efficient than _DISCONTIGMEM_.

[Reducing page structures for huge pages](https://lwn.net/Articles/839737/)

[Physical Memory Model](https://www.kernel.org/doc/html/latest/vm/memory-model.html)

Linux abstracts this diversity using one of the two memory models: FLATMEM and SPARSEMEM.

All the memory models track the status of physical page frames using struct page arranged in one or more arrays. `pfn_to_page()` and `page_to_pfn()`

_FLATMEM8_
  - The simplest memory model is FLATMEM. This model is suitable for non-NUMA systems with contiguous, or mostly contiguous, physical memory.
  - In the FLATMEM memory model, there is a global mem_map array that maps the entire physical memory.
  - To allocate the mem_map array, architecture specific setup code should call free_area_init() function. Yet, the mappings array is not usable until the call to memblock_free_all() that hands all the memory to the page allocator.
  - With FLATMEM, the conversion between a PFN and the struct page is straightforward: `PFN - ARCH_PFN_OFFSET` is an index to the `mem_map` array. The A`RCH_PFN_OFFSET` defines the first page frame number for systems with physical memory starting at address different from 0.

_SPARSEMEM_
  - SPARSEMEM is the most versatile memory model available in Linux and it is the only memory model that supports several advanced features such as hot-plug and hot-remove of the physical memory, alternative memory maps for non-volatile memory devices and deferred initialization of the memory map for larger systems.
  - The SPARSEMEM model presents the physical memory as a collection of sections. A section is represented with `struct mem_section` that contains `section_mem_map` that is, logically, a pointer to an array of `struct page`s.
  - The mem_section objects are arranged in a two-dimensional array called `mem_sections`.
  - The architecture setup code should call `sparse_init()` to initialize the memory sections and the memory maps.
  - With SPARSEMEM there are two possible ways to convert a PFN to the corresponding struct page - a _classic sparse_ and _sparse vmemmap_.
  - The classic sparse encodes the section number of a page in page->flags and uses high bits of a PFN to access the section that maps that page frame. Inside a section, the PFN is the index to the array of pages.
  - The sparse vmemmap uses a virtually mapped memory map to optimize pfn_to_page and page_to_pfn operations. There is a global struct page *vmemmap pointer that points to a virtually contiguous array of struct page objects. A PFN is an index to that array and the offset of the struct page from vmemmap is the PFN of that page.

_ZONE_DEVICE_
TO BE STUDIED  
