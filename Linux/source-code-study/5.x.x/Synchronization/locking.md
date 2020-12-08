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

#### kernel/locking/rwsem.c

```
down_write

```
