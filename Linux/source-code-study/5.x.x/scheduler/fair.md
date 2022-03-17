## Completely Fairness Scheduler

### Key Data Structures
```
struct sched_class fair_sched_class = {
	.next			= &idle_sched_class,
	.enqueue_task		= enqueue_task_fair,
	.dequeue_task		= dequeue_task_fair,
	.yield_task		= yield_task_fair,
	.yield_to_task		= yield_to_task_fair,

	.check_preempt_curr	= check_preempt_wakeup,

	.pick_next_task		= pick_next_task_fair,
	.put_prev_task		= put_prev_task_fair,
	.set_next_task          = set_next_task_fair,

	.balance		= balance_fair,
	.select_task_rq		= select_task_rq_fair,
	.migrate_task_rq	= migrate_task_rq_fair,

	.rq_online		= rq_online_fair,
	.rq_offline		= rq_offline_fair,

	.task_dead		= task_dead_fair,
	.set_cpus_allowed	= set_cpus_allowed_common,

	.task_tick		= task_tick_fair,
	.task_fork		= task_fork_fair,

	.prio_changed		= prio_changed_fair,
	.switched_from		= switched_from_fair,
	.switched_to		= switched_to_fair,

	.get_rr_interval	= get_rr_interval_fair,

	.update_curr		= update_curr_fair,

	.task_change_group	= task_change_group_fair,
};

```

```
// which CPU for the task
select_task_rq_fair
```


```
// should preempt the currently running task?
check_preempt_wakeup

```

### Init
```
__init init_sched_fair_class
  	open_softirq(SCHED_SOFTIRQ, run_rebalance_domains)
```

```
set_next_task_fair

```


```
run_rebalance_domains
  ???
```


```
sched_group_set_shares
  ???
```
