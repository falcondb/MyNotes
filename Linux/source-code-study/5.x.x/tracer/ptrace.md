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
  PTRACE_TRACEME: ptrace_traceme; arch_ptrace_attach

  PTRACE_ATTACH, PTRACE_SEIZE: ptrace_attach; arch_ptrace_attach
```

* `ptrace_attach`
```
ptrace_attach
  write_lock_irq(&tasklist_lock)
  task->ptrace = flags
  ptrace_link(task, current)
    list_add(&child->ptrace_entry, &new_parent->ptraced)
    child->parent = new_parent
  write_unlock_irq(&tasklist_lock)
```
