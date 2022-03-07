## QEMU Deep Dive

### [QEMU Internals: Overall architecture and threading model](http://blog.vmsplice.net/2011/03/qemu-internals-overall-architecture-and.html)

There are two popular architectures for programs that need to respond to events from multiple sources:
  - Parallel architecture splits work into processes or threads that can execute simultaneously. I will call this threaded architecture.
  - Event-driven architecture reacts to events by running a main loop that dispatches to event handlers.
QEMU actually uses a hybrid architecture that combines event-driven programming with threads.

#### The event-driven core of QEMU
QEMU's main event loop is `main_loop_wait` in `util/main-loop.c`:
  - Waits for file descriptors to become readable or writable. Added by `qemu_set_fd_handler`
  - Runs expired timers.Added by `qemu_mod_timer`
  - Runs bottom-halves. Added by `qemu_bh_schedule`

When a file descriptor becomes ready, a timer expires, or a BH is scheduled, the event loop invokes a callback
    - No other core code is executing at the same time so *synchronization is NOT necessary*. Callbacks execute *sequentially and atomically* with respect to other core code. There is only one thread of control executing core code at any given time.
    - No blocking system calls or long-running computations should be performed.

 `posix-aio-compat.c`, an asynchronous file I/O implementation.

 When a worker thread needs to notify core QEMU, a pipe or a `qemu_eventfd()` file descriptor is added to the event loop.  

#### Executing guest code
_Tiny Code Generator (TCG)_ and KVM

Signals are used to break out of the guest.

Timers, I/O completion, and notifications from worker threads to core QEMU use signals to ensure that the event loop will be run immediately.

#### iothread and non-iothread architecture

##### non-iothread architecture !CONFIG_IOTHREAD

The traditional architecture is a single QEMU thread that executes guest code and the event loop.

If the guest is started with multiple vcpus, no additional QEMU threads will be created, thread multiplexes instead.

##### iothread architecture CONFIG_IOTHREAD

One QEMU thread per vcpu plus a dedicated event loop thread.

`./configure --enable-io-thread`

A global mutex that synchronizes core QEMU code across the vcpus and iothread.


### [QEMU internals by Airbus](https://airbus-seclab.github.io/qemu_blog/)
