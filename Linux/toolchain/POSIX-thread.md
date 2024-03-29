## thread : Pthread -- Native POSIX Thread Library / LinuxThreads ##
* pthread_cancel
  send a cancellation request to a thread, state: enabled/disable, type:asynchronous/deferred
         1. Cancellation clean-up handlers are popped (reverse order pushed) and called.
         2. Thread-specific data destructors are called
         3. The thread is terminated.
         4. pthread_join
* pthread_testcancel  
  request delivery of any pending cancellation request         

* pthread_setcancelstate, pthread_setcanceltype:
       PTHREAD_CANCEL_ENABLE, PTHREAD_CANCEL_DISABLE,  PTHREAD_CANCEL_DEFERRED, PTHREAD_CANCEL_ASYNCHRONOUS

* pthread_cleanup_push,  pthread_cleanup_pop - push/pop thread cancellation clean-up handlers
> The calls pthread_cleanup_pop() and pthread_cleanup_push() are
typically implemented as macros emphasizing that they must be paired (one
push, one pop) at the same lexical level. Jumping out of the push/pop block may
compile, but would leave the cleanup handlers on the stack. Jumping into a block
will probably crash the program as soon as the pop is executed. Don’t do this.

* pthread_cleanup_push_defer_np,  pthread_cleanup_pop_restore_np - push/pop clean-up handlers, saving cancelability type, change to defer
* pthread_sigmask - examine and change mask of blocked signals
* pthread_kill  -  send a signal to a thread

* pthread_barrier_wait
* pthread_keycreate
* pthread_get/setspecific

`errno is a thread-specific data`

* int pthread_atfork(void (*prepare)(void), void (*parent)(void), void (*child)(void));
> pthread_atfork() shall declare fork handlers to be called before and after fork(), in the context of the thread that called fork().
The prepare fork handler shall be called before fork() processing commences.
The parent fork handle shall be called after fork() processing completes in the parent process.
The child fork handler shall be called after fork() processing completes in the child process

### Further References
* [thread-safe/Async-cancel-safe/Cancellation points](http://man7.org/linux/man-pages/man7/pthreads.7.html)
* pthread primer
* [POSIX thread scheduling APIs](http://man7.org/linux/man-pages/man7/sched.7.html)
* Keep the CPU cache warm in multiple-CPUs system.

### Notes

* Don’t Wait for Threads
* Avoid cancellation if at all possible
* Don't count same signal. The same signal could be filtered out in the queue of signal handle.
* Designate one thread to do all of the signal handling for your program.
* There are two ways of designating a signal-handling thread. You can mask out all
asynchronous signals on all threads but one, then let that one thread run your
signal handlers. You can just create the thread, and then immediately have it
block. Even though it’s sleeping, the library will still wake it up to run the signal
handler. The other, more recommended method, is to have this one thread call
sigwait().

* Thread vs light weight process vs CPU assignment
* Thread owns stack and stack pointer; a program counter; some thread information, such as scheduling priority,
and signal mask, stored in the thread structure; and the CPU registers. Everything else comes either from the process or (in a few cases) the LWP.
* A lightweight process can be thought of as a virtual CPU that is available for
executing code.
* LWPs are an implementation technique for providing kernel-level concurrency and parallelism to support the threads interface

### Concurrency vs parallel
* Concurrency: multiple threads on a shared CPU; parallel: multiple threads on mulitple CPUs.
* Context-switch triggers: Synchronization, Preemption, Yielding, Time-Slicing

* Preemption vs Time-Slicing
    * Preemption requires a system call, so the kernel has to send the signal, which
takes time.
    * Finally the LWP, to which the signal is directed, has to receive it and
run the signal handler.

* Mutex is not a cancellation point, but semaphore and condition variable (suggestion: mutex clean-up in cancellation hanlder) are.

* Purposes of POSIX signal: error reporting, situation reporting, and interruption.

* the other thread lib is ISO thread lib
