## Preemption

### Enable & disable preemption
[Proper Locking Under a Preemptible Kernel: Keeping Kernel Code Preempt-Safe](https://www.kernel.org/doc/Documentation/preempt-locking.txt)
- RULE #1: Per-CPU data structures need explicit protection
- RULE #2: CPU state must be protected
- RULE #3: Lock acquire and release must be performed by same task

- `preempt_enable/disable` are nestable.
- Any `cond_resched()` or `cond_resched_lock()` might trigger a reschedule if the preempt count is 0.
