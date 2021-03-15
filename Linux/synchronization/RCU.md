## RCU

### RCU concepts and internal
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

[What is RCU? Part 2: Usage](https://lwn.net/Articles/263130/)
Advantages of RCU include performance, deadlock immunity, and realtime latency.

Although RCU offers significant performance advantages for read-mostly workloads, one of the primary reasons for creating RCU in the first place was in fact its immunity to read-side deadlocks. This immunity stems from the fact that RCU read-side primitives do not block, spin, or even do backwards branches, so that their execution time is deterministic. It is therefore impossible for them to participate in a deadlock cycle.

An interesting consequence of RCU's read-side deadlock immunity is that it is possible to unconditionally upgrade an RCU reader to an RCU updater. Attempting to do such an upgrade with reader-writer locking results in deadlock.

RCU is susceptible to more subtle priority-inversion scenarios, for example, a high-priority process blocked waiting for an RCU grace period to elapse can be blocked by low-priority RCU readers in -rt kernels. This can be solved by using RCU priority boosting.

Because RCU readers never spin nor block, and because updaters are not subject to any sort of rollback or abort semantics, RCU readers and updaters must necessarily run concurrently.

Because RCU updaters can make changes without waiting for RCU readers to finish, the RCU readers might well see the change more quickly than would batch-fair reader-writer-locking readers

Reader-writer locking and RCU simply provide different guarantees. With reader-writer locking, any reader that begins after the writer starts executing is guaranteed to see new values, and readers that attempt to start while the writer is spinning might or might not see new values, depending on the reader/writer preference of the rwlock implementation in question. In contrast, with RCU, any reader that begins after the updater completes is guaranteed to see new values, and readers that end after the updater begins might or might not see new values, depending on timing.

although reader-writer locking does indeed guarantee consistency within the confines of the computer system, there are situations where this consistency comes at the price of increased inconsistency with the outside world. In other words, reader-writer locking obtains internal consistency at the price of silently stale data with respect to the outside world.

**RCU is a Restricted Reference-Counting Mechanism**
Overview of Linux-Kernel Reference Counting [PDF].

**RCU is a Bulk Reference-Counting Mechanism**
RCU's light-weight read-side primitives permit extremely frequent read-side usage with negligible performance degradation, permitting RCU to be used as a "bulk reference-counting" mechanism with little or no performance penalty.

**RCU is a Poor Man's Garbage Collector**
RCU differs from a GC in that: (1) the programmer must manually indicate when a given data structure is eligible to be collected, and (2) the programmer must manually mark the RCU read-side critical sections where references might legitimately be held.



[Linux Kernel Doc: What is RCU](https://www.kernel.org/doc/html/latest/RCU/whatisRCU.html)
The basic idea behind RCU is to split updates into “removal” and “reclamation” phases. The reason that it is safe to run the removal phase concurrently with readers is the semantics of modern CPUs guarantee that readers will see either the old or the new version of the data structure rather than a partially updated reference. the reclamation phase must not start until readers no longer hold references to those data items. Splitting the update into removal and reclamation phases permits the updater to perform the removal phase immediately, and to defer the reclamation phase until all readers active during the removal phase have completed, either by blocking until they finish or by registering a callback that is invoked after they finish.

### Basic APIs
So the typical RCU update sequence goes something like the following:
  - Remove pointers to a data structure, so that subsequent readers cannot gain a reference to it.
  - Wait for all previous readers to complete their RCU read-side critical sections.
  - At this point, there cannot be any readers who hold references to the data structure, so it now may safely be reclaimed

RCU-based updaters typically take advantage of the fact that writes to single aligned pointers are atomic on modern CPUs.

* `rcu_read_lock()`
Used by a reader to inform the reclaimer that the reader is entering an RCU read-side critical section. Any RCU-protected data structure accessed during an RCU read-side critical section is guaranteed to remain unreclaimed for the full duration of that critical section.

* `rcu_read_unlock()`
Used by a reader to inform the reclaimer that the reader is exiting an RCU read-side critical section. RCU read-side critical sections may be nested and/or overlapping.

* `synchronize_rcu()`
Marks the end of updater code and the beginning of reclaimer code. It does this by blocking until all pre-existing RCU read-side critical sections on all CPUs have completed. Many RCU implementations process requests in batches in order to improve efficiencies, which can further delay `synchronize_rcu()`

* `call_rcu()`
a callback form of `synchronize_rcu()`. It registers a function and argument which are invoked after all ongoing RCU read-side critical sections have completed. This callback variant is particularly useful in situations where it is illegal to block or where update-side performance is critically important. The function invokes func(head) after a grace period has elapsed. This invocation might happen from either softirq or process context, so the function is not permitted to block. If the callback for `call_rcu()` is not doing anything more than calling `kfree()` on the structure, you can use `kfree_rcu()` instead of `call_rcu()`

* `rcu_assign_pointer()`
The updater uses this function to assign a new value to an RCU-protected pointer, in order to safely communicate the change in value from the updater to the reader. It does execute any memory-barrier instructions required for a given CPU architecture

* `rcu_dereference()`
The reader uses rcu_dereference() to fetch an RCU-protected pointer, which returns a value that may then be safely dereferenced. Note that rcu_dereference() does not actually dereference the pointer, instead, it protects the pointer for later dereferencing. It also executes any needed memory-barrier instructions for a given CPU architecture.

### Toy implementations
* `Read-write lock`
`synchronize_rcu` acquires write lock first and release it before a memory barrier call.
Synchronization calls' overhead, possible dead lock

* `CLASSIC RCU`
`rcu_read_lock` and `rcu_read_unlock` do nothing, `synchronize_rcu` makes sure it is scheduled on each CPU. It is illegal to block while in an RCU read-side critical section.


[Introduction to RCU homepage](http://www.rdrop.com/users/paulmck/RCU/)
[Resizable, Scalable, Concurrent Hash Tables via Relativistic Programming](https://www.usenix.org/legacy/event/atc11/tech/final_files/Triplett.pdf)

[Review Checklist for RCU Patches](https://www.kernel.org/doc/Documentation/RCU/checklist.txt)
- Is RCU being applied to a read-mostly situation?  If the data structure is updated more than about 10% of the time, then you should strongly consider some other approach
 Splitting the update into removal and reclamation phases permits the updater to perform the removal phase immediately, and to defer the reclamation phase until all readers active during the removal phase have completed, either by blocking until they finish or by registering a callback that is invoked after they finish.
 So the typical RCU update sequence goes something like the following:

Remove pointers to a data structure, so that subsequent readers cannot gain a reference to it.
Wait for all previous readers to complete their RCU read-side critical sections.
At this point, there cannot be any readers who hold references to the data structure, so it now may safely be reclaimed

[RCU on Wikipedia](https://en.wikipedia.org/wiki/Read-copy-update)
RCU is in stark contrast with more traditional synchronization primitives such as locking or transactions that coordinate in time, but not in space

Any statement that is not within an RCU read-side critical section is said to be in a _quiescent state_. Any time period during which each thread resides at least once in a quiescent state is called a _grace period_.


### RCU APIs
[RCU API table](https://lwn.net/Articles/419086/)

[RCU part 3: the RCU API](https://lwn.net/Articles/264090/)
The asynchronous update-side primitive, `call_rcu()`, invokes a specified function with a specified argument after a subsequent grace period. For example, `call_rcu(p,f)`; will result in the RCU callback `f(p)` being invoked after a subsequent grace period.

When it is necessary to wait for all outstanding RCU callbacks to complete. The `rcu_barrier()` primitive does this job.

The `rcu_assign_pointer()` primitive ensures that any prior initialization remains **ordered before the assignment to the pointer on weakly ordered machines**. the `rcu_dereference()` primitive ensures that subsequent code dereferencing the pointer will see the effects of initialization code prior to the corresponding `rcu_assign_pointer()` on Alpha CPUs.

[The RCU API, 2010 Edition](https://lwn.net/Articles/418853/)


[The RCU API, 2014 Edition](https://lwn.net/Articles/609904/)
The new implementation offers much lower-latency grace periods (which was important for KVM), and, unlike other RCU implementations, allows readers in the idle loop and even in offline CPUs.

Another important addition is kfree_rcu(), which allows “fire and forget” freeing of RCU-protected data.

Given a structure p with an `rcu_head` field imaginatively named `rh`, you can now free a structure pointed to by `p` as follows: `kfree_rcu(p, rh)`

A new bitlocked linked list (`hlist_bl_head and hlist_bl_node`) was added. Bitlocked linked lists required the RCU-safe accessors `hlist_bl_for_each_entry_rcu(), hlist_bl_first_rcu(), hlist_bl_add_head_rcu(), hlist_bl_del_rcu(), hlist_bl_del_init_rcu(), and hlist_bl_set_first_rcu()`.

Performance issues in networking led to the addition of `RCU_INIT_POINTER()`, which can be used in place of `rcu_assign_pointer()` in a few special cases, and that omits `rcu_assign_pointer()`'s barrier and volatile cast. Ugly-code issues led to the addition of `RCU_POINTER_INITIALIZER()`, which may be used to initialize RCU-protected pointers in structures at compile time.

TO BE CONTAINED

[The RCU API, 2019 edition](https://lwn.net/Articles/777036/)

### RCU examples
[Using RCU to Protect Read-Mostly Linked Lists](https://www.kernel.org/doc/html/latest/RCU/listRCU.html)
- Example 1: Read-mostly list: Deferred Destruction
```
write_lock(&tasklist_lock);
list_del_rcu(&p->tasks);
write_unlock(&tasklist_lock);
call_rcu(&p->rcu, delayed_put_task_struct);
```
- Example 2: Read-Side Action Taken Outside of Lock: No In-Place Updates
```
list_del_rcu(&e->list);
call_rcu(&e->rcu, audit_free_rule);
```

- Example 3: Handling In-Place Updates
```
ne = kmalloc(sizeof(*entry), GFP_ATOMIC);
copy_rule(&ne->rule, &e->rule);
ne->rule.action = newaction;
list_replace_rcu(&e->list, &ne->list);
call_rcu(&e->rcu, audit_free_rule);
```

- Example 4: Eliminating Stale Data
```
spin_lock(&e->lock);
list_del_rcu(&e->list);
e->deleted = 1;
spin_unlock(&e->lock);
call_rcu(&e->rcu, audit_free_rule);
```
Q: For the deleted-flag technique to be helpful, why is it necessary to hold the per-entry lock while returning from the search function?
A: If the search function drops the per-entry lock before returning, then the caller will be processing stale data in any case.

- Example 5: Skipping Stale Objects


### RCU data structure
[RCU's major data structures](https://www.kernel.org/doc/Documentation/RCU/Design/Data-Structures/Data-Structures.html)
Each leaf node of the rcu_node tree has up to 16 rcu_data structures associated with it, so that there are NR_CPUS number of rcu_data structures, one for each possible CPU.

The purpose of this combining tree is to allow per-CPU events such as quiescent states, dyntick-idle transitions, and CPU hotplug operations to be processed efficiently and scalably.

Quiescent states are recorded by the per-CPU rcu_data structures, and other events are recorded by the leaf-level rcu_node structures.

All of these events are combined at each level of the tree until finally grace periods are completed at the tree's root rcu_node structure. A grace period can be completed at the root once every CPU has passed through a quiescent state. Once a grace period has completed, record of that fact is propagated back down the tree.

In effect, the combining tree acts like a big shock absorber, keeping lock contention under control at all tree levels regardless of the level of loading on the system.



[Verification of the Tree-Based Hierarchical Read-Copy Update in the Linux Kernel](https://arxiv.org/pdf/1610.03052.pdf)
