# Linux Kernel Development

## Linux kernel design patterns
[Linux kernel design patterns, part 1](https://lwn.net/Articles/336224/)

[Linux kernel design patterns, part 2](https://lwn.net/Articles/336255/)

[Linux kernel design patterns, part 3](https://lwn.net/Articles/336262/)

## System Call

[Anatomy of a system call, part 1](https://lwn.net/Articles/604287/)
The SYSCALL_DEFINEn micro; Syscall table for call number;
x86_64 syscall invocation: sys_call_table with kernel function addresses, RAX for the function number in the table
wrmsrl instruction writes to model-specific register,  MSR_LSTAR in x86_64, copy registers to kernel registers and
call the function address

[Anatomy of a system call, part 2](https://lwn.net/Articles/604515/)
Anatomy of system call implementation in the x86 32 and 64 architecture.
Introduction of vDSO, unfortunately the [post](http://www.trilithium.com/johan/2005/08/linux-gate/) by Johan Petersson is not available any more.
A little bit about ptrace(): syscall tracing

[Creating a vDSO: the Colonel's Other Chicken](https://www.linuxjournal.com/content/creating-vdso-colonels-other-chicken)
It is a step-by-step introduction of adding vDSO function to userspace and kernel space, but the code is hard to follow and reading the Linux source code probably can release the magic
