## Scheduler BPF

### [Controlling the CPU scheduler with BPF @LWN](https://lwn.net/Articles/873244/)

### [PATCH of Scheduler BPF](https://lwn.net/ml/linux-kernel/20210916162451.709260-1-guro@fb.com/)

a few observations and ideas:
- Different workloads want different policies. Being able to configure
  the policy per workload could be useful.
- A workload that benefits from not being preempted itself could still
  benefit from preempting (low priority) background system tasks.
- It would be useful to quickly (and safely) experiment with different
  policies in production, without having to shut down applications or reboot
  systems, to determine what the policies for different workloads should be.
- Only a few workloads are large and sensitive enough to merit their own
  policy tweaks. CFS by itself should be good enough for everything else,
  and we probably do not want policy tweaks to be a replacement for anything
  CFS does.


This patch adds 3 hooks to control wakeup and tick preemption:
  - `cfs_check_preempt_tick`
  - `cfs_check_preempt_wakeup`
  - `cfs_wakeup_preempt_entity`

  - The first one allows to *force or suppress a preemption* from a *tick context*. An obvious usage example is to minimize the number of non-voluntary context switches and decrease an associated latency penalty by (conditionally) providing tasks or task groups an extended execution slice. It can be used instead of tweaking `sysctl_sched_min_granularity`.

  - The second one is called from the *wakeup preemption* code and *allows to redefine whether a newly woken task should preempt the execution of the current task*. This is useful to minimize a number of preemptions of latency sensitive tasks. To some extent it's a more flexible analog of a `sysctl_sched_wakeup_granularity`.

  - The third one is similar, but it tweaks the `wakeup_preempt_entity()` function, which is called not only from a *wakeup context*, but also from *`pick_next_task()`*, which allows to *influence the decision on which task will be running next*.


## TO BE READ
[Scale wakeup granularity relative to nr_running](https://lore.kernel.org/lkml/20210920142614.4891-3-mgorman@techsingularity.net/)
[newidle_balance/sched_migration_cost](https://lore.kernel.org/lkml/ef3b3e55-8be9-595f-6d54-886d13a7e2fd@hisilicon.com/)
[The many faces of "latency nice"](https://lwn.net/Articles/820659/)
