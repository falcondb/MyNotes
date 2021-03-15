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

[The Definitive Guide to Linux System Calls](https://blog.packagecloud.io/eng/2016/04/05/the-definitive-guide-to-linux-system-calls/)
A software interrupt is raised by executing a piece of code. On x86-64 systems, a software interrupt can be raised by executing the int instruction.
use the CPU instructions `rdmsr` to `wrmsr` to read and write _MSRs_ (Model Specific Registers)
```
sudo apt-get install msr-tools
sudo modprobe msr
sudo rdmsr
```
* Legacy system calls
  - `IA32_SYSCALL_VECTOR` is a defined as 0x80 in `arch/x86/include/asm/irq_vectors.h`
  - Kernel-side: `int $0x80` entry point in `arch/x86/ia32/ia32entry.S`
    ```
    ia32_do_call:
        IA32_ARG_FIXUP
        call *ia32_sys_call_table(,%rax,8) # xxx: rip relative
    ```
  - Systemcall defined in `arch/x86/syscalls/syscall_32.tbl`
  - Intel `iret` instruction:
  > As with a real-address mode interrupt return, the IRET instruction pops the return instruction pointer, return code segment selector, and EFLAGS image from the stack to the EIP, CS, and EFLAGS registers, respectively, and then resumes execution of the interrupted program or procedure.

* 32-bit fast system calls `sysenter/sysexit`
  - Intel instruction set reference:
  > Prior to executing the SYSENTER instruction, software must specify the privilege level 0 code segment and code entry point, and the privilege level 0 stack segment and stack pointer by writing values to the following MSRs:
    >> IA32_SYSENTER_CS (MSR address 174H) — The lower 16 bits of this MSR are the segment selector for the privilege level 0 code segment. This value is also used to determine the segment selector of the privilege level 0 stack segment (see the Operation section). This value cannot indicate a null selector.
    >> IA32_SYSENTER_EIP (MSR address 176H) — The value of this MSR is loaded into RIP (thus, this value references the first instruction of the selected operating procedure or routine). In protected mode, only bits 31:0 are loaded.
    >> IA32_SYSENTER_ESP (MSR address 175H) — The value of this MSR is loaded into RSP (thus, this value contains the stack pointer for the privilege level 0 stack). This value cannot represent a non-canonical address. In protected mode, only bits 31:0 are loaded.  

  - Housekeeping work in wrapper function, `__kernel_vsyscall` in `arch/x86/vdso/vdso32/sysenter.S`
  - Kernel-side: sysenter entry point in `arch/x86/ia32/ia32entry.S`:
  ```
  sysenter_dispatch:
        call    *ia32_sys_call_table(,%rax,8)
  ```      
  - Returning from a sysenter system call with sysexit
    The kernel can use the sysexit instruction to resume execution back to the user program. The caller is expected to put the address to return to into the rdx register, and to put the pointer to the program stack to use in the rcx register.
  - the assembly code in the section `sysexit_from_sys_call` of `arch/x86/ia32/ia32entry.S`

* 64-bit fast system calls `syscall/sysret`
  - Intel instruction set reference:
  > SYSCALL invokes an OS system-call handler at privilege level 0. It does so by loading RIP from the IA32_LSTAR MSR (after saving the address of the instruction following SYSCALL into RCX).

  > User-level applications use as integer registers for passing the sequence %rdi, %rsi, %rdx, %rcx, %r8 and %r9. The kernel interface uses %rdi, %rsi, %rdx, %r10, %r8 and %r9.
  > A system-call is done via the syscall instruction. The kernel destroys registers %rcx and %r11.
  > The number of the syscall has to be passed in register %rax.
  > System-calls are limited to six arguments, no argument is passed directly on the stack.
  > Returning from the syscall, register %rax contains the result of the system-call. A value in the range between -4095 and -1 indicates an error, it is -errno.
  > Only values of class INTEGER or class MEMORY are passed to the kernel.

  - Highlight in the Intel instruction set reference: register %rax for syscall number and also the result value; up to 6 input arguments in `%rdi, %rsi, %rdx, %r10, %r8, %r9` registers

  - Kernel-side: syscall entry point
  ```
  system_call_fastpath:
    call *sys_call_table(,%rax,8)
  ```
  the syscall table defined in arch/x86/syscalls/syscall_64.tbl

  - Returning from a syscall system call with sysret
    - the address to where execution should be resume is copied into the rcx register when syscall is used.

* glibc syscall wrapper internals
  find `__kernel_vsyscall` by searching the ELF auxilliary headers to find a header with type `AT_SYSINFO` which contained the address of `__kernel_vsyscall`.

[Creating a vDSO: the Colonel's Other Chicken](https://www.linuxjournal.com/content/creating-vdso-colonels-other-chicken)
It is a step-by-step introduction of adding vDSO function to userspace and kernel space, but the code is hard to follow and reading the Linux source code probably can release the magic

[Linux Kernel Teaching](https://linux-kernel-labs.github.io/refs/heads/master/index.html)
Good lectures and examples on Syscall, interrupt, and SMP.

Disabling preemption (interrupts)
[x86 and amd64 instruction reference](https://www.felixcloutier.com/x86/index.html)
```
#define local_irq_disable() \
    asm volatile („cli” : : : „memory”)
```
`CLI` clears the IF flag in the EFLAGS register and no other flags are affected. Clearing the IF flag causes the processor to ignore maskable external interrupts.

```
#define local_irq_enable() \
   asm volatile („sti” : : : „memory”)
```
`STI` sets the interrupt flag (IF) in the EFLAGS register. This allows the processor to respond to maskable hardware interrupts

```
#define local_irq_save(flags) \
   asm volatile ("pushf ; pop %0" :"=g" (flags)
                 : /* no input */: "memory") \
   asm volatile("cli": : :"memory")
```
`pushf`: Decrements the stack pointer by 4 (if the current operand-size attribute is 32) and pushes the entire contents of the EFLAGS register onto the stack, or decrements the stack pointer by 2 (if the operand-size attribute is 16) and pushes the lower 16 bits of the EFLAGS register (that is, the FLAGS register) onto the stack. These instructions reverse the operation of the POPF/POPFD instructions.

`pop`: Loads the value from the top of the stack to the location specified with the destination operand (or explicit opcode) and then increments the stack pointer. The destination operand can be a general-purpose register, memory location, or segment register.

"=g": eax, ebx, ecx, edx or variable in memory, here is `flags`.
```
#define local_irq_restore(flags) \
   asm volatile ("push %0 ; popf"
                 : /* no output */
                 : "g" (flags) :"memory", "cc");
```
`popf`: Pops a doubleword (POPFD) from the top of the stack (if the current operand-size attribute is 32) and stores the value in the EFLAGS register, or pops a word from the top of the stack (if the operand-size attribute is 16) and stores it in the lower 16 bits of the EFLAGS register (that is, the FLAGS register). These instructions reverse the operation of the PUSHF/PUSHFD/PUSHFQ instructions.
Clobbers: `cc` flags register, `memory` memory reads or writes.

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

[Understanding the Linux Kernel Initcall Mechanism](https://kernelnewbies.org/Documents/InitcallMechanism)

[linux-insides](https://0xax.gitbooks.io/linux-insides/content/)
