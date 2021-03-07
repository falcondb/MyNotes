[What is RCU, Fundamentally?](https://lwn.net/Articles/262464/)
RCU achieves scalability improvements by allowing reads to occur concurrently with updates. In contrast with conventional locking primitives that ensure mutual exclusion among concurrent threads regardless of whether they be readers or updaters, or with reader-writer locks that allow concurrent reads but not in the presence of updates, RCU supports concurrency between a single updater and multiple readers.

RCU ensures that reads are coherent by maintaining multiple versions of objects and ensuring that they are not freed up until all pre-existing read-side critical sections complete.

RCU defines and uses efficient and scalable mechanisms for publishing and reading new versions of an object, and also for deferring the collection of old versions.

RCU is made up of three fundamental mechanisms, the first being used for insertion, the second being used for deletion, and the third being used to allow readers to tolerate concurrent insertions and deletions.

The `rcu_assign_pointer()` would publish the new structure, **forcing both the compiler and the CPU to execute the assignment to gp after the assignments to the fields referenced by p**.

The `rcu_dereference()` primitive can thus be thought of as subscribing to a given value of the specified pointer, **guaranteeing that subsequent dereference operations will see any initialization that occurred before the corresponding publish (rcu_assign_pointer()) operation**.

In its most basic form, RCU is a way of waiting for things to finish. Of course, there are a great many other ways of waiting for things to finish, including reference counts, reader-writer locks, events, and so on. The great advantage of RCU is that it can wait for each of (say) 20,000 different things without having to explicitly track each and every one of them, and without having to worry about the performance degradation, scalability limitations, complex deadlock scenarios, and memory-leak hazards that are inherent in schemes using explicit tracking.

There is a trick, and the trick is that RCU Classic read-side critical sections delimited by `rcu_read_lock()` and `rcu_read_unlock()` are **not permitted to block or sleep**. Therefore, **when a given CPU executes a context switch, we are guaranteed that any prior RCU read-side critical sections will have completed**. This means that as soon as each CPU has executed at least one context switch, all prior RCU read-side critical sections are guaranteed to have completed, meaning that synchronize_rcu() can safely return.

Thus, RCU Classic's `synchronize_rcu()` can conceptually be as simple as the following:
```
  1 for_each_online_cpu(cpu)
  2   run_on(cpu);
```  
Here, `run_on()` switches the current thread to the specified CPU, which forces a context switch on that CPU.


[Linux Kernel Doc: What is RCU](https://www.kernel.org/doc/html/latest/RCU/whatisRCU.html)
### RCU OVERVIEW
The basic idea behind RCU is to split updates into “removal” and “reclamation” phases. The reason that it is safe to run the removal phase concurrently with readers is the semantics of modern CPUs guarantee that readers will see either the old or the new version of the data structure rather than a partially updated reference. the reclamation phase must not start until readers no longer hold references to those data items. Splitting the update into removal and reclamation phases permits the updater to perform the removal phase immediately, and to defer the reclamation phase until all readers active during the removal phase have completed, either by blocking until they finish or by registering a callback that is invoked after they finish.

So the typical RCU update sequence goes something like the following:
  - Remove pointers to a data structure, so that subsequent readers cannot gain a reference to it.
  - Wait for all previous readers to complete their RCU read-side critical sections.
  - At this point, there cannot be any readers who hold references to the data structure, so it now may safely be reclaimed

RCU-based updaters typically take advantage of the fact that writes to single aligned pointers are atomic on modern CPUs.

[Introduction to RCU homepage](http://www.rdrop.com/users/paulmck/RCU/)
[Resizable, Scalable, Concurrent Hash Tables via Relativistic Programming](https://www.usenix.org/legacy/event/atc11/tech/final_files/Triplett.pdf)

[Review Checklist for RCU Patches](https://www.kernel.org/doc/Documentation/RCU/checklist.txt)
- Is RCU being applied to a read-mostly situation?  If the data structure is updated more than about 10% of the time, then you should strongly consider some other approach
 Splitting the update into removal and reclamation phases permits the updater to perform the removal phase immediately, and to defer the reclamation phase until all readers active during the removal phase have completed, either by blocking until they finish or by registering a callback that is invoked after they finish.
 So the typical RCU update sequence goes something like the following:

Remove pointers to a data structure, so that subsequent readers cannot gain a reference to it.
Wait for all previous readers to complete their RCU read-side critical sections.
At this point, there cannot be any readers who hold references to the data structure, so it now may safely be reclaimed
