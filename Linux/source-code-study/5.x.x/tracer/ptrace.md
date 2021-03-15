## ptrace
[ptrace man](https://man7.org/linux/man-pages/man2/ptrace.2.html)
A tracee first needs to be attached to the tracer.  Attachment and subsequent commands are per thread.
Ptrace commands are always sent to a specific tracee using a call of the form `ptrace(PTRACE_foo, pid, ...)`
- `PTRACE_TRACEME`: Indicate that this process is to be traced by its parent.

- `PTRACE_SYSCALL, PTRACE_SINGLESTEP`: Restart the stopped tracee as for PTRACE_CONT, but arrange for the tracee to be stopped at the next entry to or exit from a system call, or after execution of a single instruction, respectively.  (The tracee will also, as usual, be stopped upon receipt of a signal.)  From the tracer's perspective, the tracee will appear to have been stopped by receipt of a SIGTRAP.

[How does strace work?](https://blog.packagecloud.io/eng/2016/02/29/how-does-strace-work/)



## ptrace in kernel

* system call entry
`kernel/trace.c`
```
SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr, unsigned long, data)
  PTRACE_TRACEME:
    ptrace_traceme
      set_tsk_thread_flag / clear_tsk_thread_flag TIF_SYSCALL_TRACE
      wake_up_state(child, __TASK_TRACED)
    arch_ptrace_attach

  PTRACE_ATTACH, PTRACE_SEIZE: ptrace_attach; arch_ptrace_attach

  arch_ptrace eventually calls ptrace_request

```

* `ptrace_request`
```
ptrace_request
  case PTRACE_SYSCALL, PTRACE_CONT
    ptrace_resume
      set_tsk_thread_flag / clear_tsk_thread_flag TIF_SYSCALL_TRACE
      wake_up_state(child, __TASK_TRACED)
```

* `ptrace_attach`
```
ptrace_attach
  write_lock_irq(&tasklist_lock)
  task->ptrace = flags
  ptrace_link(task, current)
    // child = task; new_parent = current
    list_add(&child->ptrace_entry, &new_parent->ptraced)
    child->parent = new_parent // the original parent stays at child->real_parent
  write_unlock_irq(&tasklist_lock)

  wait_on_bit
  proc_ptrace_connector
```

## process events connector
[process events connector](https://lwn.net/Articles/157150/)

* init
```
__init cn_proc_init
  cn_add_callback(&cn_proc_event_id, "cn_proc", &cn_proc_mcast_ctl)
```

```
subsys_initcall(cn_init)
cn_init
  *dev = &cdev
  struct netlink_kernel_cfg cfg = {
  .groups	= CN_NETLINK_USERS + 0xf,
  .input	= cn_rx_skb,}
  dev->nls = netlink_kernel_create(&init_net, NETLINK_CONNECTOR, &cfg) // see netlink.md

  dev->cbdev = cn_queue_alloc_dev   // callback queue
    struct cn_queue_dev * = kzalloc
    INIT_LIST_HEAD(&dev->queue_list)
    spin_lock_init(&dev->queue_lock)
  proc_create_single  // proc fs
```

* `proc_ptrace_connector` in `drivers/connector/cn_proc.c`
```
proc_ptrace_connector
  struct cn_msg *msg
  struct proc_event *ev
  ev = msg->data

  ev->what = PROC_EVENT_PTRACE
  ev->event_data.sid.process_pid = task->pid
  ev->event_data.sid.process_tgid = task->tgid

  memcpy(&msg->id, &cn_proc_event_id, sizeof(msg->id))    // cn_rx_skb will check the msg->id to trigger registered callback
  send_msg(msg) ==> cn_netlink_send ==>  cn_netlink_send_mult

```
* ` cn_netlink_send_mult` in `drivers/connector/connector.c`
```
cn_netlink_send_mult
  struct cn_dev *dev = &cdev
  skb = nlmsg_new

  netlink_unicast / netlink_broadcast   // see netlink.md
```

```
cn_rx_skb
  cn_call_callback
    find the matched struct cn_callback_entry in dev->cbdev->queue_list
    struct cn_callback_entry -> callback
```

## Syscall tracing
In release 3.x, the code of running the actual syscall code (call the function pointer in `sys_call_table`) is `arch/x86/kernel/entry_64.S.` as assembly code. In release 5.x, the code of pushing user mode registers to stack before syscall is in `arch/x86/entry/entry_64.S:ENTRY(entry_SYSCALL_64)` and the code of making the syscall is in `arch/x86/entry/do_syscall_64.c:do_syscall_64()`.

```
do_syscall_64
  if ti->flags & _TIF_WORK_SYSCALL_ENTRY
    syscall_trace_enter(regs)
      if ti->flags & (_TIF_SYSCALL_TRACE | _TIF_SYSCALL_EMU)
        tracehook_report_syscall_entry    ==>   ptrace_report_syscall
          ptrace_notify \\ signal.c
            ptrace_do_notify
              info.si_signo = SIGTRAP
              ptrace_stop(..., &info)
                set_special_state(TASK_TRACED)

                do_notify_parent_cldstop
                  __group_send_sig_info(SIGCHLD, &info, parent) ==> send_signal // see signal.md signal is put in task's pending list

                  __wake_up_parent
                    __wake_up_parent \\ in exit.c
                      __wake_up_sync_key(&parent->signal->wait_chldexit, TASK_INTERRUPTIBLE, struct task_struct p) // sched.md

                cgroup_enter_frozen() // set task->frozen
            		preempt_enable_no_resched()
            		freezable_schedule()
            		cgroup_leave_frozen(true)

          fatal_signal_pending(SIGTRAP)
      ## SECCOMP staff  

      if test_thread_flag(TIF_SYSCALL_TRACEPOINT)
  		  trace_sys_enter(regs, regs->orig_ax)  

  regs->ax = sys_call_table[nr](regs)  
```
