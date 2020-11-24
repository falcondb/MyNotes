# Linux Kernel Development

## Documentation for binutils
[Documentation for binutils](https://sourceware.org/binutils/docs/)

## Linux kernel design patterns
[Linux kernel design patterns, part 1](https://lwn.net/Articles/336224/)

[Linux kernel design patterns, part 2](https://lwn.net/Articles/336255/)

[Linux kernel design patterns, part 3](https://lwn.net/Articles/336262/)

[Ghosts of Unix Past: a historical search for design patterns](https://lwn.net/Articles/411845/)

## System Call

[Anatomy of a system call, part 1](https://lwn.net/Articles/604287/)
x86_64 syscall invocation: sys_call_table with kernel function addresses, RAX for the function number in the table
wrmsrl instruction writes to model-specific register,  MSR_LSTAR in x86_64, copy registers to kernel registers and
call the function address
See my note about `syscall.h` in 5.x linux release.

[Anatomy of a system call, part 2](https://lwn.net/Articles/604515/)
Anatomy of system call implementation in the x86 32 and 64 architecture.
_sys_call_table_ is accessed from the _ia32_sysenter_target_ entry point of `arch/x86/kernel/entry_32.S`
The location of the _ia32_sysenter_target_ entry point gets written to a _model-specific register (MSR)_ at kernel start (`in enable_sep_cpu()`)
_MSR_IA32_SYSENTER_EIP (0x176)_ handles the _SYSENTER_ instruction
64-bit programs use the _SYSCALL_ instruction; Modern 32-bit programs use the _SYSENTER_ instruction; Ancient 32-bit programs use the `INT 0x80` instruction to trigger a software interrupt handler
Introduction of vDSO, unfortunately the [post](http://www.trilithium.com/johan/2005/08/linux-gate/) by Johan Petersson is not available any more.
A little bit about ptrace(): syscall tracing

[Creating a vDSO: the Colonel's Other Chicken](https://www.linuxjournal.com/content/creating-vdso-colonels-other-chicken)
It is a step-by-step introduction of adding vDSO function to userspace and kernel space, but the code is hard to follow and reading the Linux source code probably can release the magic

[Linux Kernel Teaching](https://linux-kernel-labs.github.io/refs/heads/master/index.html)
Good lectures and examples on Syscall, interrupt, and SMP.

Disabling preemption (interrupts)
```
## TO BE further study
#define local_irq_disable() \
    asm volatile („cli” : : : „memory”)

#define local_irq_enable() \
   asm volatile („sti” : : : „memory”)

#define local_irq_save(flags) \
   asm volatile ("pushf ; pop %0" :"=g" (flags)
                 : /* no input */: "memory") \
   asm volatile("cli": : :"memory")

#define local_irq_restore(flags) \
   asm volatile ("push %0 ; popf"
                 : /* no output */
                 : "g" (flags) :"memory", "cc");
```
>Although the interrupts can be explicitly disabled and enable with local_irq_disable() and local_irq_enable() these APIs should only be used when the current state and interrupts is known. They are usually used in core kernel code (like interrupt handling).

>For typical cases where we want to avoid interrupts due to concurrency issues it is recommended to use the local_irq_save() and local_irq_restore() variants. They take care of saving and restoring the interrupts states so they can be freely called from overlapping critical sections without the risk of accidentally enabling interrupts while still in a critical section, as long as the calls are balanced.

Cache coherency in multi-processor systems
>MESI (named after the acronym of the cache line states names: Modified, Exclusive, Shared and Invalid)

Optimized spin locks
>for the x86 architecture, the current spin lock implementation uses a queued spin lock where the CPU cores spin on different locks (hopefully distributed in different cache lines) to avoid cache invalidation operations



Process and Interrupt Context Synchronization
>* In process context: disable interrupts and acquire a spin lock; this will protect both against interrupt or other CPU cores race conditions (spin_lock_irqsave() and spin_lock_restore() combine the two operations)

>* In interrupt context: take a spin lock; this will will protect against race conditions with other interrupt handlers or process context running on different processors

>* In process context use spin_lock_bh() (which combines local_bh_disable() and spin_lock()) and spin_unlock_bh() (which combines spin_unlock() and local_bh_enable())

>* In bottom half context use: spin_lock() and spin_unlock() (or spin_lock_irqsave() and spin_lock_irqrestore() if sharing data with interrupt handlers)

```
#define PREEMPT_BITS      8
#define SOFTIRQ_BITS      8
#define HARDIRQ_BITS      4
#define NMI_BITS          1

#define preempt_disable() preempt_count_inc()

#define local_bh_disable() add_preempt_count(SOFTIRQ_OFFSET)

#define local_bh_enable() sub_preempt_count(SOFTIRQ_OFFSET)

#define irq_count() (preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK))

#define in_interrupt() irq_count()

asmlinkage void do_softirq(void)
{
    if (in_interrupt()) return;
    ...
}
```
