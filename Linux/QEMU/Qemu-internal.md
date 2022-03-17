## QEMU Deep Dive

### [The memory API @QEMU source code](https://github.com/qemu/qemu/blob/master/docs/devel/memory.rst)
- Modeling:
  - ordinary RAM
  - memory-mapped I/O
  - memory controllers

- Support:
  - tracking RAM changes by the guest
  - setting up coalesced memory for kvm
  - setting up ioeventfd regions for kvm

- Memory is modelled as an acyclic graph of MemoryRegion objects. Sinks (leaves) are RAM and MMIO regions, while other nodes represent buses, memory controllers, and memory regions that have been rerouted
- AddressSpace objects for every root and possibly for intermediate MemoryRegions

- Types of regions (`struct MemoryRegion`)
  - RAM: `memory_region_init_ram, memory_region_init_resizeable_ram, memory_region_init_ram_from_file, memory_region_init_ram_ptr`
  - MMIO: a range of guest memory that is implemented by host callbacks. `memory_region_init_io`, `MemoryRegionOps` for the callback
  - ROM: `memory_region_init_rom`
  - IOMMU region: an IOMMU region translates addresses of accesses made to it and forwards them to some other target memory region. `memory_region_init_iommu`
  - container: for grouping several regions into one unit.
  - alias: allow a region to be split apart into discontiguous regions.

- Migration
  - `memory_region_init_ram_nomigrate, memory_region_init_rom_nomigrate, memory_region_init_rom_device_nomigrate`, only initialization, but no migration

- Region lifecycle
  - `memory_region_add_subregion memory_region_del_subregion`
  - Destruction of a memory region happens automatically when the owner object dies; A dynamically allocated data structure,should call `object_unparent`
  - Okay to call object_unparent at any time for an alias or a container region

- Overlapping regions and priority
  - `memory_region_add_subregion_overlap`
  - specifies a priority that allows the core to decide which of two regions at the same address are visible (highest wins)
  - Examples of the flat view of the overlapping regions (informative examples)

- Visibility
  - all direct subregions of the root region are matched against the address, in descending priority order
    - if the address lies outside the region offset/size, the subregion is discarded
    - if the subregion is a leaf (RAM or MMIO), the search terminates, returning this leaf region
    - if the subregion is a container, the same algorithm is used within the subregion (after the address is adjusted by the subregion offset)
    - if the subregion is an alias, the search is continued at the alias target (after the address is adjusted by the subregion offset and alias offset)
    - if a recursive search within a container or alias subregion does not find a match (because of a "hole" in the container's coverage of its address range), then if this is a container with its own MMIO or RAM backing the search terminates, returning the container itself. Otherwise we continue with the next subregion in priority order
  - if none of the subregions match the address then the search terminates with no match found

- MMIO Operations


### [QEMU Internals: Overall architecture and threading model](http://blog.vmsplice.net/2011/03/qemu-internals-overall-architecture-and.html)

There are two popular architectures for programs that need to respond to events from multiple sources:
  - Parallel architecture splits work into processes or threads that can execute simultaneously. I will call this threaded architecture.
  - Event-driven architecture reacts to events by running a main loop that dispatches to event handlers.
QEMU actually uses a hybrid architecture that combines event-driven programming with threads.

#### The event-driven core of QEMU
QEMU's main event loop is `main_loop_wait` in `util/main-loop.c`:
  - Waits for file descriptors to become readable or writable. Added by `qemu_set_fd_handler`
  - Runs expired timers.Added by `qemu_mod_timer`
  - Runs bottom-halves. Added by `qemu_bh_schedule`

When a file descriptor becomes ready, a timer expires, or a BH is scheduled, the event loop invokes a callback
    - No other core code is executing at the same time so *synchronization is NOT necessary*. Callbacks execute *sequentially and atomically* with respect to other core code. There is only one thread of control executing core code at any given time.
    - No blocking system calls or long-running computations should be performed.

 `posix-aio-compat.c`, an asynchronous file I/O implementation.

 When a worker thread needs to notify core QEMU, a pipe or a `qemu_eventfd()` file descriptor is added to the event loop.  

#### Executing guest code
_Tiny Code Generator (TCG)_ and KVM

Signals are used to break out of the guest.

Timers, I/O completion, and notifications from worker threads to core QEMU use signals to ensure that the event loop will be run immediately.

#### iothread and non-iothread architecture

##### non-iothread architecture !CONFIG_IOTHREAD

The traditional architecture is a single QEMU thread that executes guest code and the event loop.

If the guest is started with multiple vcpus, no additional QEMU threads will be created, thread multiplexes instead.

##### iothread architecture CONFIG_IOTHREAD

One QEMU thread per vcpu plus a dedicated event loop thread.

`./configure --enable-io-thread`

A global mutex that synchronizes core QEMU code across the vcpus and iothread.

### [QEMU Internals: How guest physical RAM works](http://blog.vmsplice.net/2016/01/qemu-internals-how-guest-physical-ram.html)

* slots emulates DIMM hotplug
* `hw/mem/pc-dimm.c`models a DIMM
* `backends/hostmem.c` host meory that backs guest RAM

* Dirty memory tracking
  * When stores to guest RAM, this needs to be noticed:
    - Live migration
    - TCG, recompile
    - Graphics card emulation

* Address spaces
  AddressSpace contains a tree of MemoryRegioins

### [QEMU Internals: Event loops](http://blog.vmsplice.net/2020/08/qemu-internals-event-loops.html)
QEMU has several different types of threads:
  - vCPU threads that execute guest code and perform device emulation synchronously with respect to the vCPU.
  - The main loop that runs the event loops
  - IOThreads that run event loops for device emulation concurrently with vCPUs and "out-of-band" QMP monitor commands.

The key point is that vCPU threads do not run an event loop. The main loop thread and IOThreads run event loops. vCPU threads can add event sources to the main loop or IOThread event loops. Callbacks run in the main loop thread or IOThreads.

### [Coroutines in QEMU: The basics](http://blog.vmsplice.net/2014/01/coroutines-in-qemu-basics.html)

### [QEMU internals by Airbus](https://airbus-seclab.github.io/qemu_blog/)
