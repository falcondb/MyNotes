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
Overview of Linux-Kernel Reference Counting.

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

Each of the data structures (`rcu_state, rcu_node & rcu_data`) has its own synchronization:
  - Each `rcu_state` structures has a lock and a mutex, and some fields are protected by the corresponding root `rcu_node` structure's lock.
  - Each `rcu_node` structure has a spinlock.
  - The fields in `rcu_data` are private to the corresponding CPU, although a few can be read and written by other CPUs.

The general role of each of these data structures is as follows:
  - `rcu_state`: This structure forms the interconnection between the rcu_node and rcu_data structures, tracks grace periods, serves as short-term repository for callbacks orphaned by CPU-hotplug events, maintains rcu_barrier() state, tracks expedited grace-period state, and maintains state used to force quiescent states when grace periods extend too long,
  - `rcu_node`: This structure forms the combining tree that propagates quiescent-state information from the leaves to the root, and also propagates grace-period information from the root to the leaves. It provides local copies of the grace-period state in order to allow this information to be accessed in a synchronized manner without suffering the scalability limitations that would otherwise be imposed by global locking. In CONFIG_PREEMPT_RCU kernels, it manages the lists of tasks that have blocked while in their current RCU read-side critical section. In CONFIG_PREEMPT_RCU with CONFIG_RCU_BOOST, it manages the per-rcu_node priority-boosting kernel threads (kthreads) and state. Finally, it records CPU-hotplug state in order to determine which CPUs should be ignored during a given grace period.
  - `rcu_data`: This per-CPU structure is the focus of quiescent-state detection and RCU callback queuing. It also tracks its relationship to the corresponding leaf rcu_node structure to allow more-efficient propagation of quiescent states up the rcu_node combining tree. Like the rcu_node structure, it provides a local copy of the grace-period information to allow for-free synchronized access to this information from the corresponding CPU. Finally, this structure records past dyntick-idle state for the corresponding CPU and also tracks statistics.
  - `rcu_head`: This structure represents RCU callbacks, and is the only structure allocated and managed by RCU users. The rcu_head structure is normally embedded within the RCU-protected data structure.

#### rcu_state
The rcu_node tree is embedded into the `->node[]` array
RCU grace periods are numbered, and the `->gp_seq` field contains the current grace-period sequence number. The bottom two bits are the state of the current grace period, which can be zero for not yet started or one for in progress. This field is protected by the root rcu_node structure's `->lock` field.

There are `->gp_seq` fields in the rcu_node and rcu_data structures as well. The fields in the rcu_state structure represent the most current value, and those of the other structures are compared in order to detect the beginnings and ends of grace periods in a distributed fashion. The values flow from rcu_state to rcu_node to rcu_data.

#### rcu_node

The rcu_node structures' `->gp_seq` fields are the counterparts of the field of the same name in the rcu_state structure. They each may lag up to one step behind their rcu_state counterpart. If the bottom two bits of a given rcu_node structure's `->gp_seq` field is zero, then this rcu_node structure believes that RCU is idle.

The `->gp_seq` field of each rcu_node structure is updated at the beginning and the end of each grace period.

The `->gp_seq_needed` fields record the furthest-in-the-future grace period request seen by the corresponding rcu_node structure. The request is considered fulfilled when the value of the `->gp_seq` field equals or exceeds that of the `->gp_seq_needed` field.

The `->qsmask` field tracks which of this rcu_node structure's children still need to report quiescent states for the current normal grace period. Such children will have a value of 1 in their corresponding bit. Note that the leaf rcu_node structures should be thought of as having rcu_data structures as their children. Similarly, the `->expmask` field tracks which of this rcu_node structure's children still need to report quiescent states for the current expedited grace period. An expedited grace period has the same conceptual properties as a normal grace period, but the expedited implementation accepts extreme CPU overhead to obtain much lower grace-period latency, for example, consuming a few tens of microseconds worth of CPU time to reduce grace-period duration from milliseconds to tens of microseconds. The `->qsmaskinit` field tracks which of this rcu_node structure's children cover for at least one online CPU.

The `->blkd_tasks` field is a list header for the list of blocked and preempted tasks. As tasks undergo context switches within RCU read-side critical sections, their task_struct structures are enqueued (via the task_struct's `->rcu_node_entry` field) onto the head of the `->blkd_tasks` list for the leaf rcu_node structure corresponding to the CPU on which the outgoing context switch executed. As these tasks later exit their RCU read-side critical sections, they remove themselves from the list.

#### rcu_segcblist
The segments are as follows:
  - _RCU_DONE_TAIL_: Callbacks whose grace periods have elapsed. These callbacks are ready to be invoked.
  - _RCU_WAIT_TAIL_: Callbacks that are waiting for the current grace period. Note that different CPUs can have different ideas about which grace period is current, hence the `->gp_seq` field.
  - _RCU_NEXT_READY_TAIL_: Callbacks waiting for the next grace period to start.
  - _RCU_NEXT_TAIL_: Callbacks that have not yet been associated with a grace period.

The `->head` pointer references the first callback or is NULL if the list contains no callbacks (which is not the same as being empty). Each element of the `->tails[]` array references the ->next pointer of the last callback in the corresponding segment of the list, or the list's `->head` pointer if that segment and all previous segments are empty. If the corresponding segment is empty but some previous segment is not empty, then the array element is identical to its predecessor. Older callbacks are closer to the head of the list, and new callbacks are added at the tail.

The `->gp_seq[]` array records grace-period numbers corresponding to the list segments. This is what allows different CPUs to have different ideas as to which is the current grace period while still avoiding premature invocation of their callbacks. In particular, this allows CPUs that go idle for extended periods to determine which of their callbacks are ready to be invoked after reawakening.

#### rcu_data
The `->grpmask` field indicates the bit in the `->mynode->qsmask` corresponding to this rcu_data structure, and is also used when propagating quiescent states.

The `->gp_seq` field is the counterpart of the field of the same name in the rcu_state and rcu_node structures. The `->gp_seq_needed` field is the counterpart of the field of the same name in the rcu_node structure. They may each lag up to one behind their rcu_node counterparts, but in `CONFIG_NO_HZ_IDLE and CONFIG_NO_HZ_FULL` kernels can lag arbitrarily far behind for CPUs in dyntick-idle mode

The `->cpu_no_qs` flag indicates that the CPU has not yet passed through a quiescent state, while the `->core_needs_qs` flag indicates that the RCU core needs a quiescent state from the corresponding CPU.

In the absence of CPU-hotplug events, RCU callbacks are invoked by the same CPU that registered them.

The CPU advances the callbacks in its rcu_data structure whenever it notices that another RCU grace period has completed. The CPU detects the completion of an RCU grace period by noticing that the value of its rcu_data structure's `->gp_seq` field differs from that of its leaf rcu_node structure. Recall that each rcu_node structure's `->gp_seq` field is updated at the beginnings and ends of each grace period.

#### Dyntick-Idle Handling
TO BE STUDIED

#### rcu_head
The `->next` field is used to link the rcu_head structures together in the lists within the rcu_data structures. The `->func` field is a pointer to the function to be called when the callback is ready to be invoked, and this function is passed a pointer to the rcu_head structure. However, `kfree_rcu()` uses the ->func field to record the offset of the rcu_head structure within the enclosing RCU-protected data structure.



[Verification of the Tree-Based Hierarchical Read-Copy Update in the Linux Kernel](https://arxiv.org/pdf/1610.03052.pdf)

The number of readers can be large (up to the number of CPUs in non-preemptible implementations and up to the number of tasks in preemptible implementations).

When a CPU is blocking, in the idle loop, or running in user mode, all RCU read-side critical sections that were previously running on that CPU
must have finished. Each of these states is therefore called a quiescent state. After each CPU has passed through a quiescent state, the corresponding RCU grace period ends.

Tree RCU therefore instead uses a tree hierarchy of data structures, each leaf of which records quiescent states of a single CPU and propagates the information up to the root. When the root is reached, a grace period has ended. Then the grace-period information is propagated down from the root to the leaves
of the tree. Shortly after the leaf data structure of a CPU receives this information, synchronize rcu() will return.

* rcu_state
The `->gpnum` field records **the most recently started grace period**, whereas `->completed` records **the most recently ended grace period**. If the two numbers are equal, then corresponding flavor of RCU is idle. If gpnum is one greater than completed, then RCU is in the middle of a grace period. All other combinations are invalid.

* rcu_node
The `->qsmask` field indicates which of this node’s children still need to report quiescent states for the current grace period. As with rcu state, the rcu node structure has `->gpnum and ->completed` fields that have values identical to those of the enclosing rcu state structure, except at the beginnings and ends of grace periods when the new values are propagated down the tree. Each of these fields can be smaller than its rcu state counterpart by at most one.

* rcu_data
The rcu data structure detects quiescent states and handles RCU callbacks for the corresponding CPU.

The rcu data structure’s `->qs_pending` field indicates that RCU **needs a quiescent state** from the corresponding CPU, and the `->passed_quiesce` indicates that the CPU has already **passed through a quiescent state**.

* Quiescent State Detection
The non-preemptible RCU-sched flavor’s quiescent states apply to CPUs, and are user-space execution, context switch, idle, and offline state. Therefore, RCU-sched only needs to track tasks and interrupt handlers that are actually running because blocked and preempted tasks are always in quiescent states.

- Scheduling-Clock Interrupt
The `rcu_check_callbacks()` is invoked from the scheduling-clock interrupt handler, which allows RCU to periodically check whether a given busy CPU is in the usermode or idle-loop quiescent states. If the CPU is in one of these quiescent states, `rcu_check_callbacks()` invokes `rcu_sched_qs()`, which sets the per-CPU `rcu_sched_data.passed_quiesce` fields to 1.
The `rcu_check_callbacks()` function invokes `rcu_pending()` to determine whether a recent event or current condition means that RCU requires attention from this CPU. If so, `rcu_check_callbacks()` invokes raise `softirq()`

- Context-Switch Handling
The context-switch quiescent state is recorded by invoking `rcu_note_context_switch()` from `schedule()`. The `rcu_note_context_switch()` function invokes `rcu_sched_qs()` to inform RCU of the context switch, which is a quiescent state of the CPU.

* Grace Period Detection
- Softirq Handler for RCU
RCU’s busy-CPU grace period detection relies on the _RCU SOFTIRQ_ handler function `rcu_process_callbacks()`. This function first calls `rcu_check_quiescent_state()` to report recent quiescent states on the current CPU. Then `rcu_process_callbacks()` starts a new grace period if needed, and finally calls invoke `rcu_callbacks()` to invoke any callbacks whose grace period has already elapsed.

Function `rcu_check_quiescent_state()` first invokes `note_gp_changes()` to update the CPU-local rcu data structure to record the end of previous grace periods and the beginning of new grace periods. If an old grace period has ended, `rcu_advance_cbs()` is invoked to advance all callbacks, otherwise, `rcu_accelerate_cbs()` is invoked to assign a grace period to any recently arrived callbacks.

`rcu_check_quiescent_state()` checks whether `->qs` pending indicates that RCU needs a quiescent state from this CPU. If so, it checks whether `->passed_quiesce` indicates that this CPU has in fact passed through a quiescent state. If so, it invokes `rcu_report_qs_rdp()` to report that quiescent state up the combining tree.

The `rcu_report_qs_rdp()` function first verifies that the CPU has in fact detected a legitimate quiescent state for the current grace period, and under the protection of the leaf rcu node structure’s ->lock. If not, it resets quiescent-state detection and returns, thus ignoring any redundant quiescentstates belonging to some earlier grace period. Otherwise, if the `->qsmask` field indicates that RCU needs to report a quiescent state from this CPU, `rcu_accelerate_cbs()` is invoked to assign a grace-period number to any new callbacks, and then `rcu_report_qs_rnp()` is invoked to report the quiescent state to the rcu node combining tree.
The `rcu_report_qs_rnp()` function traverses up the rcu node tree, at each level holding the rcu node structure’s `->lock`. At any level, if the child structure’s `->qsmask` bit is already clear, or if the `->gpnum` changes, traversal stops. Otherwise, the child structure’s bit is cleared from `->qsmask`, after which, if `->qsmask` is non-zero, traversal stops. Otherwise, traversal proceeds on to the parent rcu node structure. Once the root is reached, traversal stops and `rcu_report_qs_rsp()` is invoked to awaken the graceperiod kthread (kernel thread). The grace-period kthread will then clean up after the now-ended grace period, and, if needed, start a new one.

- Grace-Period Kernel Thread
When no grace period is required, the grace-period kthread sets its rcu state structure’s `->flags` field to `RCU_GP WAIT GPS` and then waits within an inner infinite loop for that structure’s `->gp` state field to be set. Once set, rcu gp kthread() invokes `rcu_gp_init()` to initialize a new grace period, which rechecks the `->gp` state field. It increments rsp->gpnum by 1 to record a new grace period number. Finally, it performs a breadth-first traversal of the rcu node structures in the combining tree. For each rcu node structure rnp, we set the `rnp->qsmask` to indicate which children must report quiescent states for the new grace period, and set `rnp->gpnum` and `rnp->completed` to their rcu state counterparts. If the rcu node structure rnp is the parent of the current CPU’s rcu data, we invoke `__note_gp_changes()` to set up the CPU-local rcu data state. Other CPUs will invoke `__note_gp_changes()` after their next scheduling-clock interrupt.
To clean up after a grace period, `rcu_gp_kthread()` calls `rcu_gp_cleanup()` after setting the rcu state field `rsp->gp` state to `RCU_GP_CLEANUP`. After the function returns, `rsp->gp` state is set to `RCU_GP_CLEANED` to record the end of the old grace period. It first sets each rcu node structure’s `->completed` field to the rcu state structure’s `->gpnum` field. It then updates the current CPU’s CPU-local rcu data structure by calling `__note_gp_changes()`. For other CPUs, the update will take place when they handle the scheduling-clock interrupts. After the traversal, it marks the completion of the grace period by setting the rcu state structure’s `->completed` field to that structure’s `->gpnum` field, and invokes `rcu_advance_cbs()` to advance callbacks. Finally, if another grace period is needed, we set `rsp->gp` flags to `RCU_GP_FLAG_INIT`.
