[LINUX KERNEL MEMORY BARRIERS](https://www.kernel.org/doc/Documentation/memory-barriers.txt)
* Memory barrier
> As can be seen above, independent memory operations are effectively performed in random order, but this can be a problem for CPU-CPU interaction and for I/O. What is required is some way of intervening to instruct the compiler and the CPU to restrict the order. Memory barriers are such interventions.  They impose a perceived partial ordering over the memory operations on either side of the barrier.

  - Write memory barriers
    >  A write memory barrier gives a guarantee that all the STORE operations specified before the barrier will appear to happen before all the STORE operations specified after the barrier with respect to the other components of the system.

  - Data dependency barriers
    > A data dependency barrier is a weaker form of read barrier.  In the case  where two loads are performed such that the second depends on the result of the first (eg: the first load retrieves the address to which the second load will be directed), a data dependency barrier would be required to make sure that the target of the second load is updated after the address obtained by the first load is accessed.

  - Read memory barriers
    > A read barrier is a data dependency barrier plus a guarantee that all the LOAD operations specified before the barrier will appear to happen before all the LOAD operations specified after the barrier with respect to the other components of the system.

  - General memory barriers
    > A general memory barrier gives a guarantee that all the LOAD and STORE operations specified before the barrier will appear to happen before all the LOAD and STORE operations specified after the barrier with respect to the other components of the system.

  - ACQUIRE operations
    > It guarantees that all memory operations after the ACQUIRE operation will appear to happen after the ACQUIRE operation with respect to the other components of the system.

  - RELEASE operations
    > It guarantees that all memory operations before the RELEASE operation will appear to happen before the RELEASE operation with respect to the other components of the   system.

> Memory barriers are only required where there's a possibility of interaction between two CPUs or between a CPU and a device.



[Explanation of the Linux-Kernel Memory Consistency Model](linux/tools/memory-model/Documentation/explanation.txt)

[Linux-Kernel Memory Model](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4444.html)

[A formal kernel memory-ordering model](https://lwn.net/Articles/718628/)

[Who's afraid of a big bad optimizing compiler?](https://lwn.net/Articles/793253/)

[ACCESS_ONCE()](https://lwn.net/Articles/508991/)

[Kernel Concurrency Sanitizer (KCSAN)](https://lwn.net/Articles/800298/)

[Dmitry Vyukov homepage](https://www.1024cores.net/)
