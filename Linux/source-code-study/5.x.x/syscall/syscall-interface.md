## System calls
Refer [a little obsolete LWN post](https://lwn.net/Articles/604287/) for system calls
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
