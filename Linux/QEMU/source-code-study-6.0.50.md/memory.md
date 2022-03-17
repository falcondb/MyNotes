## Memory management related code

### Key data structures

#### RAMBlock in `exec/ramblock.h`

* RAMBlock: Host mmap memory chunks backed up by mmap
```
struct RAMBlock {
  struct rcu_head rcu
  struct MemoryRegion *mr
  uint8_t *host

  void (*resized)(...)

  // dirty bitmap
  unsigned long *bmap;
  // bitmap of already received pages in postcopy
  unsigned long *receivedmap;
  // READ the comment in the source code!!!
  unsigned long *clear_bmap;
```

#### Global list of RAMBlock
```
RAMList ram_list = { .blocks = QLIST_HEAD_INITIALIZER(ram_list.blocks) }
```

#### memory region in `exec/memory.h`
```
struct MemoryRegion {
  RAMBlock *ram_block
  MemoryRegionOps *ops

  MemoryRegion *container

  QTAILQ_HEAD(, MemoryRegion) subregions;
  QTAILQ_ENTRY(MemoryRegion) subregions_link;
  QTAILQ_HEAD(, CoalescedMemoryRange) coalesced;

   MemoryRegionIoeventfd *ioeventfds

}


struct IOMMUMemoryRegion {
    MemoryRegion parent_obj;
    QLIST_HEAD(, IOMMUNotifier) iommu_notify;
}
```

```
// callbacks structure for updates to the physical memory map
struct MemoryListener {
  void (*begin)()
  void (*commit)()
  void (*region_add)
  void (*region_del)
  void (*region_nop)
  ...
}
```

```
//  a mapping of addresses to MemoryRegion objects
struct AddressSpace {
  MemoryRegion *root
  struct MemoryRegionIoeventfd *ioeventfds;
  QTAILQ_HEAD(, MemoryListener) listeners;
  QTAILQ_ENTRY(AddressSpace) address_spaces_link;
```


### `backends/hostmem.c`


### `softmmu/physmem.c`

```
qemu_ram_alloc
  qemu_ram_alloc_internal
    ram_block_add(RAMBlock *new_block)
```

```
qemu_ram_alloc_from_fd

```

```
ram_block_add
  new_block->offset = find_ram_offset()
    // TO BE STUDIED

  if !new_block->host
      if !xen
        new_block->host = qemu_anon_ram_alloc
            qemu_ram_mmap
               mmap

        memory_try_enable_merging

  cpu_physical_memory_set_dirty_range      

  ram_block_notify_add

```


```
address_space_rw

```

### `exec.c`
```
cpu_physical_memory_rw

```


### `softmmu/memory.c`
```
memory_region_init_ram_shared_nomigrate
  MemoryRegion->ram_blcok = qemu_ram_alloc
```

### `util/mmap-alloc.c`
```
qemu_ram_mmap

  mmap
```

### `hw/boards.h`
```
memory_region_allocate_system_memory
  ????
```
