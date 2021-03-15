[Linux Kernel Doc: What is RCU](https://www.kernel.org/doc/html/latest/RCU/whatisRCU.html)
### RCU OVERVIEW
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
