## Native Linux KVM tool

### On LWN.net
[Native Linux KVM tool](https://lwn.net/Articles/436781/)

### Source code
At `tools/kvm/`

### Key Executions

#### setup
```
kvm_cmd_setup ==> do_setup
  make_dir(guestfs_name) // make the root fs for the guest system
kvm->cfg
  copy_init     // Copy the root image to virt/init of the guest system as an initramfs

  copy_passwd   // set root user in /etc/passwd of the guest system
```

#### run
```
kvm_cmd_run
  kvm_cmd_run_init
    // read cmd options and set up kvm->cfg
    // call the struct init_item.init() registered in init_lists. see util/init.c util-init.h
  kvm_cmd_run_work
    foreach kvm->nrcpus
      pthread_create(&kvm->cpus[i]->thread, NULL, kvm_cpu_thread, kvm->cpus[i])
  kvm_cmd_run_exit
    init_list__exit
```

```
kvm_cpu_thread
  kvm_cpu__start        // kvm-cpu.c
    // block SIGALAM
    signal(SIGKVMEXIT, kvm_cpu_signal_handler)
    signal(SIGKVMPAUSE, kvm_cpu_signal_handler)
    kvm_cpu__reset_vcpu  // reset virtual CPU to a known state

    while cpu->is_running


      kvm_cpu__run  ==>  ioctl(vcpu->vcpu_fd, KVM_RUN, 0)

      switch cpu->kvm_run->exit_reason
        case KVM_EXIT_IO:
          kvm_cpu__emulate_io

        case KVM_EXIT_MMIO:
          kvm_cpu__handle_coalesced_mmio
          kvm_cpu__emulate_mmio


```

#### `kvm/ioport.c`
```
kvm_cpu__emulate_io   // arch dependent
  kvm__emulate_io   ==>   kvm__emulate_io
    entry = ioport_search(&ioport_tree, port)     // ioport_tree: a RB tree of struct ioport, port is from kvm_run.io for KVM_EXIT_IO_IN / KVM_EXIT_IO_OUT
    ops = entry->ops
    ops->io_in / ops->io_out  // repeat it for the given number of blocks
```

```
ioport__register
  *entry = (struct ioport) {
		.node		= RB_INT_INIT(port, port + count),
		.ops		= ops,
		.priv		= param,
		.dev_hdr	= (struct device_header) {
			.bus_type	= DEVICE_BUS_IOPORT,
			.data		= generate_ioport_fdt_node,
		},
	}

  ioport_insert(&ioport_tree, entry)

  device__register(&entry->dev_hdr)
```

#### `device.c`
```
device__register
  bus = &device_trees[dev->bus_type]
  dev->dev_num = bus->dev_num++

  pci__assign_irq OR virtio_mmio_assign_irq
    irq__alloc_line()  ==>  next_line++ // assign an IRQ line

  // insert the device to the device RB tree by device number

```


#### `ioeventfd.c`
```
ioeventfd__init
  epoll_event = {.events = EPOLLIN}
  epoll_fd = epoll_create
  epoll_stop_fd = eventfd
  epoll_event.data.fd = epoll_stop_fd
  epoll_ctl(epoll_fd, EPOLL_CTL_ADD, epoll_stop_fd, &epoll_event)

  ioeventfd__start
    pthread_create(&thread, NULL, ioeventfd__thread, NULL)
```

```
static void *ioeventfd__thread

	for
		nfds = epoll_wait(epoll_fd, events, IOEVENTFD_MAX_EVENTS, -1);
		for i < nfds
			if events[i].data.fd == epoll_stop_fd
				goto done;

			ioevent = events[i].data.ptr;

			read(ioevent->fd, &tmp, sizeof(tmp)
			ioevent->fn(ioevent->fn_kvm, ioevent->fn_ptr);

done:
    write(epoll_stop_fd, &tmp, sizeof(tmp));
```

```
ioeventfd__add_event
    kvm_ioevent = (struct kvm_ioeventfd) {
      .addr		= ioevent->io_addr,
      .len		= ioevent->io_len,
      .datamatch	= ioevent->datamatch,
      .fd		= event,
      .flags		= KVM_IOEVENTFD_FLAG_DATAMATCH,
    }

    ioctl(ioevent->fn_kvm->vm_fd, KVM_IOEVENTFD, &kvm_ioevent)

    epoll_event = (struct epoll_event) {
      .events		= EPOLLIN,
      .data.ptr	= new_ioevent,
    }

    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, event, &epoll_event)

    list_add_tail(&new_ioevent->list, &used_ioevents)
```
