## Qemu migration

### [Migration @QEMU source code](https://github.com/qemu/qemu/blob/master/docs/devel/migration.rst)
  - Transports
    - TCP, Unix, exec(stdin/stdout), fd, RDMA
  - Debugging
    - qemu console: `migrate "exec:cat > mig"`; ./scripts/analyze-migration.py -f mig

  - VMState

  - Iterative device migration
    - use `qemu_put_* / qemu_get_*` in `migration/qemu-file-types.h` to read/write data from/to the stream
    - An iterative device must provide the following functions:
      - `save_setup, load_setup`
      - `save_live_pending`: how much more data the iterative data must save
      - `save_live_iterate`: send a chunk of data
      - `save_live_complete_precopy`: transmit the last section for the device
      - `load_state`: load sections
      - `cleanup`: save and load that are called at the end of migration

  - Stream structure
    - see the [format description](https://github.com/qemu/qemu/blob/master/docs/devel/migration.rst#stream-structure)

  - Postcopy
    - *Postcopy* migration is a way to deal with migrations that refuse to converge.
    - its plus side is that there is an upper bound on the amount of migration traffic and time
    - the down side is that during the postcopy phase, a failure of either side or the network connection causes the guest to be lost
    - In postcopy the destination CPUs are started before all the memory has been transferred, and accesses to pages that are yet to be transferred cause a fault that's translated by QEMU into a request to the source QEMU.
    - *Postcopy* can be combined with *precopy* so that if *precopy* doesn't finish in a given time the switch is made to *postcopy*.

  - Enabling postcopy
    - `migrate_set_capability postcopy-ram on` to enable postcopy
    - `migrate_start_postcopy` start the normal migration in precopy mode
    - `migrate_start_postcopy` switch from precopy to postcopy
    -
    - `migrate_set_capability postcopy-blocktime on` to enable *blcoktime* accounting, which shows how long the vCPU was in state of interruptible sleep due to pagefault. `query-migrate qmp` command to retrieve it.

    - Source behaviour
      - Until postcopy is entered the migration stream is identical to normal precopy, except for the addition of a 'postcopy advise' command at the beginning
      - When postcopy starts the source sends the page discard data and then forms the 'package' containing:
        - Command: 'postcopy listen'
        - The device state
        - A series of sections, identical to the precopy streams device state stream containing everything except postcopiable devices
        - Command: 'postcopy run'
      - The 'package' is sent as the data part of a Command: *CMD_PACKAGED*
      - During postcopy the source scans the list of dirty pages and sends them
      - when a page request is received from the destination, the dirty page scanning restarts from the requested location

    - Destination behaviour
      -

    - Postcopy states
      - Advise, Discard, Listen, Running, End
      - Details in the [original doc](https://github.com/qemu/qemu/blob/master/docs/devel/migration.rst#postcopy-states)

    - Source side page maps
      - *the migration bitmap* and *unsent map*
      - The *unsent map* is used for the transition to postcopy.
        - Have been sent but then redirtied (which must be discarded)
        - Have not yet been sent - which also must be discarded to cause any transparent huge pages built during precopy to be broken.

    - Postcopy with hugepages
      - The linux kernel on the destination must support userfault on hugepages.
      - The huge-page configuration on the source and destination VMs must be identical
      - Note that -mem-path /dev/hugepages will fall back to allocating normal RAM if it doesn't have enough hugepages. Using -mem-prealloc enforces the allocation using hugepages.
      - Care should be taken with the size of hugepage used; until the full page is transferred the destination thread is blocked.

    - Postcopy with shared memory
      - The Linux kernel userfault support works on /dev/shm memory and on hugetlbfs
      - *NEED TO READ IT AGAIN*

### [Live Migrating QEMU-KVM Virtual Machines](https://developers.redhat.com/blog/2015/03/24/live-migrating-qemu-kvm-virtual-machines#)
  - QEMU Layout
    - the guest-visible device state, includes the device state for each device that has been exposed to the guest
    - QEMU state that is not involved during the migration process, but it is quite important to get this right before migration is started

  - Getting Configuration Right
    - *REFER TO* the details in the original doc: storage, wall-clock, network, cpu, machine-type in QEMU

  - Stages in Live Migration
    - 1 Mark all RAM dirty; 2 Keep sending dirty RAM pages; 3 Stop guest, transfer remaining dirty RAM, device state
