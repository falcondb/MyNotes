## source code related to the main loop

### `softmmu/main.c`
```
main
  qemu_init
    // see its section

  qemu_main_loop
    // see its section
    // I am studying here!!!!

  qemu_cleanup

```


### `softmmu/vl.c`

```
qemu_init
  qemu_init_subsystems
  // see its section

  // process the cmd options, e.g., call drive_add()

  qemu_init_main_loop

  cpu_timers_init

  qemu_create_machine

  qemu_create_default_devices
  qemu_create_early_backends

  configure_accelerators
    // nothing other than figuring out which accelerator (kvm, tcg, etc) to use

  migration_object_init

  qemu_create_late_backends
  qemu_init_displays
  accel_setup_post
  os_setup_post

```

```
qemu_create_machine

  //softmmu/physmem.c
  cpu_exec_init_all
    io_mem_init
      memory_region_init_io(&io_mem_unassigned, NULL, &unassigned_mem_ops, ...)
        // softmmu/memory.c
        memory_region_init
          // populate MemoryRegion
          memory_region_do_init

    memory_map_init
      // populate MemoryRegion
      memory_region_init(system_memory, "system")
      address_space_init(&address_space_memory, system_memory, "memory")
        // populate AddressSpace
        address_space_update_ioeventfds

      memory_region_init_io(system_io, &unassigned_io_ops, "io")
      address_space_init(&address_space_io, system_io, "I/O")

  page_size_init  // adjust page size, cpu.c
```



### util/main-loop.c
```
qemu_init_main_loop

  init_clocks(qemu_timer_notify_cb)
    for QEMU_CLOCK_MAX  qemu_clock_init(type, qemu_timer_notify_cb)
      qemu_timer_notify_cb = qemu_notify_event ==> qemu_bh_schedule(qemu_notify_bh)


```

```
main_loop_wait

  notifier_list_notify(&main_loop_poll_notifiers, &mlpoll //MainLoopPoll)

  ret = os_host_main_loop_wait(timeout_ns)
  mlpoll.state = ret < 0 ? MAIN_LOOP_POLL_ERR : MAIN_LOOP_POLL_OK
  notifier_list_notify(&main_loop_poll_notifiers, &mlpoll)

  qemu_clock_run_all_timers
    for type < QEMU_CLOCK_MAX
        if (qemu_clock_use_for_deadline(type))
          qemu_clock_run_timers(type);
            ts->cb(ts->opaque)

```

```
os_host_main_loop_wait
  // Don't quit follow the poll fds, need more study
  qemu_mutex_unlock_iothread()
  replay_mutex_unlock()

  qemu_poll_ns(gpollfds->data, ...)

  replay_mutex_lock()
  qemu_mutex_lock_iothread()

  glib_pollfds_poll()
```

### util/qemu-timer.c
```
timerlist_run_timers  ==> timerlist_run_timers

```


### softmmu/runstate
```
qemu_init_subsystems

    qemu_init_cpu_list
      qemu_mutex_init(&qemu_cpu_list_lock)
      qemu_cond_init(&exclusive_cond, &exclusive_resume, &qemu_work_cond)

    qemu_init_cpu_loop
      // See its section

    qemu_mutex_lock_iothread  ==> qemu_mutex_lock_iothread_impl

    runstate_init
      qemu_mutex_init(&vmstop_lock)
```

* [A deep dive into QEMU: VM running states](https://airbus-seclab.github.io/qemu_blog/runstate.html)

```
qemu_main_loop
   while !main_loop_should_exit()
     main_loop_wait(false)
```

```
main_loop_should_exit

  if qemu_debug_requested
    vm_stop(RUN_STATE_DEBUG)

  if qemu_suspend_requested
    qemu_system_suspend
      pause_all_vcpus
      notifier_list_notify(&suspend_notifiers)
      runstate_set(RUN_STATE_SUSPENDED)
      qapi_event_send_suspend()

  if qemu_shutdown_requested
    qemu_system_shutdown
      qapi_event_send_shutdown
      notifier_list_notify(&shutdown_notifiers)

  if qemu_reset_requested
    pause_all_vcpus
    qemu_system_reset
      MachineClass->reset()  
    resume_all_vcpus
      FOREACH cpu: cpu_resume

  if qemu_wakeup_requested
    pause_all_vcpus
    qemu_system_wakeup
      MachineClass->wakeup()
    notifier_list_notify(&wakeup_notifiers, ...)
    resume_all_vcpus()
    qapi_event_send_wakeup()

  if qemu_powerdown_requested
    qemu_system_powerdown
      qapi_event_send_powerdown
      notifier_list_notify(&powerdown_notifiers)

  if qemu_vmstop_requested
    vm_stop

```



### util/notify.c
```
notifier_list_notify
  QLIST_FOREACH_SAFE(list->notifiers)
    notifier->notify(notifier, data)

```
