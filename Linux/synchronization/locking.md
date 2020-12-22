[Locking lessons](linux/Documentation/locking/spinlocks.rst)


[Linux Device Driver - Chapter 5 Concurrency and Race
Conditions](https://static.lwn.net/images/pdf/LDD3/ch05.pdf)
> Here is the hard rule of resource sharing: any time that a hardware or software resource is shared beyond a single thread of execution, and the possibility exists that
one thread could encounter an inconsistent view of that resource, you must explicitly manage access to that resource.

> no object can be made available to the kernel until it is in a state where it can function properly, and references to such objects must be tracked.

> critical sections: code that can be executed by only one thread at any given time.

`void sema_init(struct semaphore *sem, int val) in linux/semaphore.h`

> In the Linux world, the P function is called down

```
# linux/semaphore.h
extern void down(struct semaphore *sem);
extern int __must_check down_interruptible(struct semaphore *sem);
extern int __must_check down_killable(struct semaphore *sem);
extern int __must_check down_trylock(struct semaphore *sem);
extern int __must_check down_timeout(struct semaphore *sem, long jiffies);
extern void up(struct semaphore *sem);
```

* Read Write Lock
```
# rwsem.h/c
void init_rwsem(struct rw_semaphore *sem);
void down_read(struct rw_semaphore *sem);
int down_read_trylock(struct rw_semaphore *sem);
void up_read(struct rw_semaphore *sem);
void down_write(struct rw_semaphore *sem);
int down_write_trylock(struct rw_semaphore *sem);
void up_write(struct rw_semaphore *sem);
void downgrade_write(struct rw_semaphore *sem);
```

`down` decrements the value of the semaphore and waits as long as need be.
`down_interruptible` does the same, but the operation is interruptible.

`down_interruptible` allows a user-space process that is waiting on a semaphore to be interrupted by the user. Noninterruptible operations are a good way to create unkillable processes

> If the operation is interrupted, the function returns a nonzero value, and the caller does not hold the semaphore

`down_trylock` never sleeps; if the semaphore is not available at the time of the call, down_trylock returns immediately with a nonzero return value.


* Spinlocks
> Unlike semaphores, spinlocks may be used in code that cannot sleep, such as interrupt handlers.

> the core rule that applies to spinlocks is that any code must, while holding a spinlock, be atomic. It cannot sleep; in fact, it cannot relinquish the processor for any reason except to service interrupts. The kernel preemption case is handled by the spinlock code itself. Any time kernel code holds a spinlock, preemption is disabled on the relevant processor.

> Sleeps can happen in surprising places; writing code that will execute under a spinlockrequires paying attention to every function that you call.

> spin_lock_irqsave disables interrupts (on the local processor only) before taking the spinlock; the previous interrupt state is stored in flags.

> If you are absolutely sure nothing else might have already disabled interrupts on your processor (or, in other words, you are sure that you should enable
interrupts when you release your spinlock), you can use spin_lock_irq instead and not have to keep track of the flags. Finally, spin_lock_bh disables software interrupts
before taking the lock, but leaves hardware interrupts enabled.


* Reader/Writer Spinlocks
```
rwlock_t my_rwlock = RW_LOCK_UNLOCKED; /* Static way */
Or
rwlock_t my_rwlock;
rwlock_init(&my_rwlock); /* Dynamic way */

void read_lock(rwlock_t *lock);
void read_lock_irqsave(rwlock_t *lock, unsigned long flags);
void read_lock_irq(rwlock_t *lock);
void read_lock_bh(rwlock_t *lock);

```

* Ambiguous Rules
> When you create a resource that can be accessed concurrently, you should define which lockwill control that access. Locking should really be laid out at the
beginning; it can be a hard thing to retrofit in afterward

> To make your locking work properly, you have to write some functions with the assumption that their caller has already acquired the relevant lock(s). Usually, only
your internal, static functions can be written in this way; functions called from outside must handle locking explicitly.

> when multiple locks must be acquired, they should always be acquired in the same order.

> If you must obtain a lockthat is local to your code (a device lock, say) along with a lock belonging to a more central part of the
kernel, take your lock first. If you have a combination of semaphores and spinlocks, you must, of course, obtain the semaphore(s) first; calling down (which can sleep)
while holding a spinlockis a serious error. But most of all, try to avoid situations where you need more than one lock.

> As a general rule, you should start with relatively coarse locking unless you have a real reason to believe that contention could be a problem. Resist the urge to
optimize prematurely; the real performance constraints often show up in unexpected places.

> If you do suspect that lockcontention is hurting performance, you may find the lockmeter tool useful.

* Lock-Free Algorithms
  -   circular buffer
    > When carefully implemented, a circular buffer requires no locking in the absence of multiple producers or consumers. The producer is the only thread that is allowed to
    modify the write index and the array location it points to. As long as the writer stores a new value into the buffer before updating the write index, the reader will always
    see a consistent view. The reader, in turn, is the only thread that can access the read index and the value it points to. With a bit of care to ensure that the two pointers do not overrun each other, the producer and the consumer can access the buffer concurrently with no race conditions.

  - Atomic Variables
    > An atomic_t (linux/types.h) holds an int value on all supported architectures

  [On atomic types](linux/Documentation/atomic_t.txt)

  - Bit Operations
    > the kernel offers a set of functions that modify or test single bits atomically. Because the whole operation happens in a single step, no interrupt
    > Atomic bit operations are very fast, since they perform the operation using a single machine instruction without disabling interrupts whenever the underlying platform
can do that.

* Seqlocks
> Seqlocks work in situations where the resource to be protected is small, simple, and frequently accessed, and where write access is rare but must be fast. Essentially, they workby allowing readers free access to the resource but requiring those readers to check for collisions with writers and, when such a collision happens, retry their access.

> Read access works by obtaining an (unsigned) integer sequence value on entry into the critical section. On exit, that sequence value is compared with the current value;
if there is a mismatch, the read access must be retried.

```
do {
  seq = read_seqbegin(&the_lock);
  /* Do what you need to do */
} while read_seqretry(&the_lock, seq);
```

> If your seqlockmight be accessed from an interrupt handler, you should use the IRQ-safe versions instead:
```
read_seqbegin_irqsave
read_seqretry_irqrestore
```

> The write_sequnlock() write lockis implemented with a spinlock, so all the usual constraints apply

* Read-Copy-Update
> RCU places a number of constraints on the sort of data structure that it can protect.
  >> It is optimized for situations where reads are common and writes are rare.
  >> The resources being protected should be accessed via pointers, and all references to those resources must be held only by atomic code.

> The code that executes while the read “lock” is held must be atomic. No reference to the protected resource may be used after the call to rcu_read_unlock.  

> once every processor on the system has been scheduled at least once, all references must be gone. So that is what RCU does; it sets aside a callback
that waits until all processors have scheduled; that callbackis then run to perform the cleanup work.

> Code that changes an RCU-protected data structure must get its cleanup callback by allocating a struct rcu_head, although it doesn’t need to initialize that structure in
any way. After the change to that resource is complete, a call should be made to `call_rcu`

```
# linux/types.h
struct callback_head {
	struct callback_head *next;
	void (*func)(struct callback_head *head);
} __attribute__((aligned(sizeof(void *))));
#define rcu_head callback_head
```

[Unreliable Guide To Locking](linux/Documentation/kernel-hacking/locking.rst)
[Unreliable Guide To Locking](https://www.kernel.org/doc/html/v4.15/kernel-hacking/locking.html#cheat-sheet-for-locking)
`spin_lock_bh` disables softirqs on that CPU. `spin_lock_irq()` and `spin_lock_irqsave()` stop hardware interrupts as well

The same softirq can run on the other CPUs: you can use a per-CPU array (see Per-CPU Data) for better performance. Use `spin_lock()` and `spin_unlock()` for shared data

If a hardware irq handler shares data with a softirq, you have two concerns. Firstly, the softirq processing can be interrupted by a hardware interrupt, and secondly, the critical region could be entered by a hardware interrupt on another CPU. This is where `spin_lock_irq()` is used. It is defined to disable interrupts on that cpu, then grab the lock.

`spin_lock_irqsave()` is architecture-specific whether all interrupts are disabled inside irq handlers themselves.

Pete Zaitcev gives the following summary:

* If you are in a process context (any syscall) and want to lock other process out, use a mutex. You can take a mutex and sleep (copy_from_user*( or kmalloc(x,GFP_KERNEL)).
* Otherwise (== data can be touched in an interrupt), use spin_lock_irqsave() and spin_unlock_irqrestore().
* Avoid holding spinlock for more than 5 lines of code and across any function call (except accessors like readb()).

`spin_trylock()` does not spin but returns non-zero if it acquires the spinlock on the first try or 0 if not

[The Linux Networking Architecture: Design and Implementation of Network Protocols in the Linux Kernel](https://freecomputerbooks.com/The-Linux-Networking-Architecture.html)
`spin_lock` tries to set the spinlock my_spinlock . If it is not free, then we have to wait or test until it is released. The free spinlock is then set immediately.

`spin_lock_irqsave` additionally _prevents interrupts_ and _stores the current value of the CPU status register_ in the variable flags.

`spin_lock_irq` _does not store the value of the CPU status register_. It assumes that interrupts are already being prevented.

`spin_lock_bh` tries to set the lock, but it _prevents bottom halfs_ from running at the same time.

#### kernel/locking/rwsem.c

```
down_write

```
