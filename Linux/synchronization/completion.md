[Completions - "wait for completion" barrier APIs](linux/Documentation/scheduler/completion.rst)
- Introduction
  If you have one or more threads that must wait for some kernel activity to have reached a point or a specific state, completions can provide a race-free solution to this problem.

  The advantage of using completions is that they have a well defined, focused purpose which makes it very easy to see the intent of the code, but they also result in more efficient code as all threads can continue execution until the result is actually needed, and both the waiting and the signaling is highly efficient using low level scheduler sleep/wakeup facilities.

  Completions are built on top of the waitqueue and wakeup infrastructure of the Linux scheduler. The event the threads on the waitqueue are waiting for is reduced to a simple flag in 'struct completion', appropriately called "done".

- Usage
  - the initialization of the `struct completion` synchronization object
  - the waiting part through a call to one of the variants of `wait_for_completion()`
  - the signaling side through a call to `complete() or complete_all()`.

  ```
  struct completion {
    unsigned int done;
    wait_queue_head_t wait;
  };
  ```
  This provides the `->wait` waitqueue to place tasks on for waiting (if any), and the `->done` completion flag for indicating whether it's completed or not.

  Initializing of dynamically allocated completion objects is done via a call to `init_completion(&dynamic_object->done);`

  `reinit_completion()`

  For static (or global) declarations
  ```
  DECLARE_COMPLETION()::

  	static DECLARE_COMPLETION(setup_done);
  	DECLARE_COMPLETION(setup_done);
  ```
  declared as a local variable within a function
  ```
  DECLARE_COMPLETION_ONSTACK(setup_done)
  ```

  `wait_for_completion(struct completion *done)`

  Note that wait_for_completion() is calling spin_lock_irq()/spin_unlock_irq(), so it can only be called safely when you know that interrupts are enabled. Calling it from IRQs-off atomic contexts will result in hard-to-detect spurious enabling of interrupts.

  The default behavior is to wait without a timeout and to mark the task as uninterruptible. wait_for_completion() and its variants are only safe in process context (as they can sleep) but not in atomic context, interrupt context, with disabled IRQs, or preemption is disabled

  `void complete(struct completion *done)` signal exactly one waiter
  `void complete_all(struct completion *done)` signal all

  Both `complete()` and `complete_all()` can be called in IRQ/atomic context safely

  `bool try_wait_for_completion(struct completion *done)` and `bool completion_done(struct completion *done)`

# kernel/sched/completion.c
```
