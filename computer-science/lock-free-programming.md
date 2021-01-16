[The Art of Multiprocessor Programming](https://www.amazon.com/Art-Multiprocessor-Programming-Revised-Reprint/dp/0123973376)

[Concurrent Reading and Writing](https://lamport.azurewebsites.net/pubs/rd-wr.pdf)

[How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs](https://lamport.azurewebsites.net/pubs/lamport-how-to-make.pdf)

[A pragmatic implementation of non-blocking linked-lists](https://www.cs.purdue.edu/homes/xyzhang/fall14/lock_free_set.pdf)

[Scalable Lock-Free Dynamic Memory Allocation](https://www.cs.tufts.edu/~nr/cs257/archive/maged-michael/pldi-2004.pdf)


[Lock-Free Programming slides](https://www.cs.cmu.edu/~410-s05/lectures/L31_LockFree.pdf)
  – Deadlock
    Processes that cannot proceed because they are waiting for resources that are held by processes that are waiting for
  – Priority Inversion
    Low-priority processes hold a lock required by a higher priority process
  – Convoying
    Several processes need locks in a roughly similar order. One slow process gets in first. All the other processes slow to the speed of the first one
  – Async-signal-safety
    Signal handlers can’t use lock-based primitives
  – Kill-tolerant availability
    If threads are killed/crash while holding locks, what happens?
  – Pre-emption tolerance
    What happens if you’re pre-empted holding a lock?
  – Overall performance
    Efficient lock-based algorithms exist
    Constant struggle between simplicity and efficiency
[Some notes on lock-free and wait-free algorithms](http://www.rossbencina.com/code/lockfree?q=~rossb/code/lockfree/)

[An Introduction to Lock-Free Programming](https://preshing.com/20120612/an-introduction-to-lock-free-programming/)
_Read-modify-write (RMW)_ allowing you to perform more complex transactions atomically. They’re especially useful when a lock-free algorithm must support multiple writers, because when multiple threads attempt an RMW on the same address, they’ll effectively line up in a row and execute those operations one-at-a-time.

On today’s architectures, the tools to enforce correct memory ordering generally fall into three categories, which prevent both compiler reordering and processor reordering:
  - A lightweight sync or fence instruction
  - A full memory fence instruction
  - Memory operations which provide acquire or release semantics.
