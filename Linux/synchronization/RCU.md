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
