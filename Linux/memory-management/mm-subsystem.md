# Memory Management subsystem

## Memory anatomy
[What every programmer should know about memory series on LWN.net](https://lwn.net/Articles/250967/)
[What every programmer should know about memory PDF version](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)

***
[KS2012: The memcg/mm minisummit](https://lwn.net/Articles/516439/)

***
[Memory Management Reference](https://www.memorymanagement.org/)

## RCU
[What is RCU, Fundamentally](https://lwn.net/Articles/262464/)

[The design of preemptible read-copy-update](https://lwn.net/Articles/253651/)

[What is RCU, Really?](http://www.rdrop.com/users/paulmck/RCU/whatisRCU.html)

[Introduction to RCU](http://www2.rdrop.com/users/paulmck/RCU/)

[A Tour Through RCU's Requirements](https://www.kernel.org/doc/Documentation/RCU/Design/Requirements/Requirements.html)

## vDSO
[Creating a vDSO: the Colonel's Other Chicken](https://www.linuxjournal.com/content/creating-vdso-colonels-other-chicken)

It is a step-by-step introduction of adding vDSO function to userspace and kernel space, but the code is hard to follow and reading the Linux source code probably can release the magic

[On vsyscalls and the vDSO](https://lwn.net/Articles/446528/)
The _vsyscall_ and _vDSO_ segments are two mechanisms used to accelerate certain system calls in Linux. Recently _vsyscall_ has come to be seen as an enabler of security attacks, so some patches have been put together to phase it out.

The _vsyscall_ area is the older of these two mechanisms. It was added as a way to execute specific system calls which do not need any real level of privilege to run. The classic example is `gettimeofday()`, the kernel allows the page containing the current time to be mapped read-only into user space; that page also contains a fast `gettimeofday()` implementation.

_vsyscall_ has some limitations; among other things, there is only space for a handful of virtual system calls. As those limitations were hit, the kernel developers introduced the more flexible vDSO implementation.

Note that the vDSO area has moved, while the vsyscall page remains at the same location. The location of the vsyscall page is nailed down in the kernel ABI, but the vDSO area - like most other areas in the user-space memory layout - has its location randomized every time it is mapped.

Andrew Lutomirski's patches for static vsyscall page and this discussion on the patches and the configuration naming issue.

## KASLR
[Kernel address space layout randomization](https://lwn.net/Articles/569635/)

##Kernel stacks on x86-64 bit
[Kernel stacks on x86-64 bit](https://www.kernel.org/doc/Documentation/x86/kernel-stacks)

>Kernel thread stacks are THREAD_SIZE (2*PAGE_SIZE) big

>specialized stacks associated with each CPU.  These stacks are only used while the kernel is in control on that CPU
>>Interrupt stack: Used for external hardware interrupts
>>The interrupt stack is also used when processing a softirq.

>>Interrupt Stack Table (IST): The IST code is an index into the Task State Segment (TSS)
>>When an interrupt occurs and the hardware loads such a descriptor, the hardware automatically sets the new stack pointer based on the IST value, then invokes the interrupt handler.
>>the interrupt came from user mode, then the interrupt handler prologue will switch back to the per-thread stack.

>The currently assigned IST stacks are :
>>* DOUBLEFAULT_STACK: interrupt 8 - Double Fault Exception (#DF). Invoked when handling one exception causes another exception
>>* NMI_STACK: non-maskable interrupts (NMI). Using IST for NMI events avoids making assumptions about the previous state of the kernel stack.
>>* DEBUG_STACK: hardware debug interrupts (interrupt 1) and for software debug interrupts (INT3)
>>* MCE_STACK: interrupt 18 - Machine Check Exception (#MC).
>
>[Understanding The Linux Memory Manager](https://www.kernel.org/doc/gorman/html/understand/)
>* Describing Physical Memory
>   - Nodes
>   - Zone
>   - Page
>   
