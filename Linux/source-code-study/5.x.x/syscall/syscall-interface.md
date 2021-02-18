## System calls
Refer [a little obsolete LWN post](https://lwn.net/Articles/604287/) for system calls

[x86 instructions](https://www.felixcloutier.com/x86/index.html)
* `SYSCALL`
SYSCALL invokes an OS system-call handler at privilege level 0. It does so by loading RIP from the IA32_LSTAR MSR.
SYSCALL also saves RFLAGS into R11 and then masks RFLAGS using the IA32_FMASK MSR; specifically, the processor clears in RFLAGS every bit corresponding to a bit that is set in the IA32_FMASK MSR. 

The CS and SS descriptor caches are not loaded from the descriptors (in GDT or LDT) referenced by those selectors. Instead, the descriptor caches are loaded with fixed values. It is the responsibility of OS software to ensure that the descriptors (in GDT or LDT) referenced by those selector values correspond to the fixed values loaded into the descriptor caches

The SYSCALL instruction does not save the stack pointer (RSP). If the OS system-call handler will change the stack pointer, it is the responsibility of software to save the previous value of the stack pointer.

* `SYSENTER`
Executes a fast call to a level 0 system procedure or routine. SYSENTER is a companion instruction to SYSEXIT. The instruction is optimized to provide the maximum performance for system calls from user code running at privilege level 3 to operating system or executive procedures running at privilege level 0.

When executed in IA-32e mode, the SYSENTER instruction transitions the logical processor to 64-bit mode; otherwise, the logical processor remains in protected mode.

Prior to executing the SYSENTER instruction, software must specify the privilege level 0 code segment and code entry point, and the privilege level 0 stack segment and stack pointer by writing values to the following MSRs:

IA32_SYSENTER_CS (MSR address 174H) — The lower 16 bits of this MSR are the segment selector for the privilege level 0 code segment. This value is also used to determine the segment selector of the privilege level 0 stack segment (see the Operation section). This value cannot indicate a null selector.
IA32_SYSENTER_EIP (MSR address 176H) — The value of this MSR is loaded into RIP (thus, this value references the first instruction of the selected operating procedure or routine). In protected mode, only bits 31:0 are loaded.
IA32_SYSENTER_ESP (MSR address 175H) — The value of this MSR is loaded into RSP (thus, this value contains the stack pointer for the privilege level 0 stack). This value cannot represent a non-canonical address. In protected mode, only bits 31:0 are loaded.
These MSRs can be read from and written to using RDMSR/WRMSR. The WRMSR instruction ensures that the IA32_SYSENTER_EIP and IA32_SYSENTER_ESP MSRs always contain canonical addresses.

The SYSENTER instruction can be invoked from all operating modes except real-address mode.

To use the SYSENTER and SYSEXIT instructions as companion instructions for transitions between privilege level 3 code and privilege level 0 operating system procedures, the following conventions must be followed:

	- The segment descriptors for the privilege level 0 code and stack segments and for the privilege level 3 code and stack segments must be contiguous in a descriptor table. This convention allows the processor to compute the segment selectors from the value entered in the SYSENTER_CS_MSR MSR.
	- The fast system call “stub” routines executed by user code (typically in shared libraries or DLLs) must save the required return IP and processor state information if a return to the calling procedure is required. Likewise, the operating system or executive procedures called with SYSENTER instructions must have access to and use this saved return and state information when returning to the user code.


#### linux/syscalls.h
```
#define SYSCALL_DEFINEx(x, sname, ...)				\
	SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
```
`__SYSCALL_DEFINEx` a recursive definition
`SYSCALL_METADATA` defines metadata of the system calls
```
*types_##sname[]
*args_##sname[]
struct syscall_metadata
SYSCALL_TRACE_ENTER_EVENT
SYSCALL_TRACE_EXIT_EVENT
__attribute__((section("__syscalls_metadata")
```

The Trace metadata for system call
```
static struct trace_event_call
__attribute__((section("_ftrace_events")))
```

The concrete Syscall definition
``#define __SYSCALL_DEFINEx(x, name, ...)``

	The wrapper function for syscall table and populate the system call program array
	``asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))	``
	The wrapper function `sys##name` is aliased as `__se_sys##name`
	``__attribute__((alias(__stringify(__se_sys##name))));	``

	``ALLOW_ERROR_INJECTION(sys##name, ERRNO);
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));\
	asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\ 		``

	The entroy point of syscall in kernel, which calls `__do_sys##name`
	```
	asmlinkage long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
	{								\
		long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));\
		__MAP(x,__SC_TEST,__VA_ARGS__);				\
		__PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));	\
		return ret;						\
	}								\
		__diag_pop();							\
	```
	The implementation of the syscall start from here
	```
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
	```
