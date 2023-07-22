## BPF extensible scheduler class
- [The extensible scheduler class](https://lwn.net/Articles/922405/)

### patch
- [patch on LWN.net](https://lwn.net/ml/linux-kernel/20221130082313.3241517-1-tj@kernel.org/)

### key ops in BPF struct ops


### key bpf helper function
- scx_bpf_consume: Transfer a task from a dsq to the current CPU's local dsq
  - dispatch (move) a task to local/remote cpu's local dsq
```
 cctx = this_cpu_ptr(&consume_ctx)
 consume_dispatch_q(cctx)
    for task in dsq->fifo
      if local cpu
      list_move_tail(&p->scx.dsq_node, &scx_rq->local_dsq.fifo);
      if remote cpu
      move_task_to_local_dsq

```
- scx_bpf_dispatch
  - if there is a direct dispatch task, dispatch it immediately, done.
  - add a dispatch_buf_ent with the task and queue id to the per CPU `dispatch_buf`

```
idx = __this_cpu_read(dispatch_buf_cursor);

this_cpu_ptr(dispatch_buf)[idx] = (struct dispatch_buf_ent){
  .task = p,
  .qseq = atomic64_read(&p->scx.ops_state) & SCX_OPSS_QSEQ_MASK,
  .dsq_id = dsq_id,
  .enq_flags = enq_flags,
};
```

### key functions in kernel

- dispatch_enqueue
  - add the task to cpu local queue
  - preempt current task if necessary

```
if (enq_flags & (SCX_ENQ_HEAD | SCX_ENQ_PREEMPT))
  list_add(&p->scx.dsq_node, &dsq->fifo);
else
  list_add_tail(&p->scx.dsq_node, &dsq->fifo);

if ((enq_flags & SCX_ENQ_PREEMPT) && p != rq->curr &&
    rq->curr->sched_class == &ext_sched_class) {
  rq->curr->scx.slice = 0;
  preempt = true;
}

if (preempt || sched_class_above(&ext_sched_class,
         rq->curr->sched_class))
  resched_curr(rq);

```

\\- do_enqueue_task
```
scx_ops.enqueue
return
dispatch_enqueue
```

- kick_cpus_irq_workfn
```
for_each_cpu(cpu, this_rq->scx.cpus_to_kick)
  resched_curr
```


### functions implement scheduler class

- task_tick_scx
```
if !curr->scx.slice
  resched_curr
```

- select_task_rq_scx
```
scx_ops.select_cpu
or
scx_select_cpu_dfl

```

- pick_next_task_scx
```
pick_task_scx
  pick first task from rq->scx.local_dsq.fifo

```

- put_prev_task_scx
```
if SCX_TASK_BAL_KEEP // balance decided to keep current
  dispatch_enqueue(&rq->scx.local_dsq, p, SCX_ENQ_HEAD)

if SCX_TASK_QUEUED
  dispatch_enqueue(local_dsq, SCX_ENQ_HEAD)
```


- balance_scx

```
if runnable and has slice or lcoal_dsq is no empty, do nothing

scx_ops.consume
or
consume_dispatch_q

if still no task in local dsq


```
cond_resched

balance()
sticky_cpu

how the queue in used?
do_enqueue_task

set_next_task_scx
