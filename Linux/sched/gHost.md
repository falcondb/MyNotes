Ghost papers
CPU scheduling challenge in coordinating userspace thread scheduler in runtime (Goland) and OS native thread scheduler. Ghost enables this coordination

Ghost scheduler is asynchronous in contrast with synchronous SCX, offline scheduling decision making (ML)

across multiple CPUs scheduling policy

consist view of kernel scheduling events from userspace schedule making (decision with obsolete seq number fails)

decisions in a transaction


how to support Cgroup
ghost + network stack
ghost + BPF


- Building gHost kernel is not very challenging. However, building ghost-userspace libs and example schedulers is a little tricky (using Google Bazel). Can you provide us some technical supports when we are blocked? Any online chatting app like Slack? Github issue is not very efficient.

- CONFIG_SCHED_CLASS_GHOST (CONFIG_DEBUG_GHOST) is the only configuration required for gHost?

- CONFIG_SWITCHTO_API? ghost_switchto, where is referenced?

- why call get_task_struct, doesn't use the returned value

- _ghost_commit_pending_txn ghost_set_pnt_state

BPF
  - BPF hooks
    - BPF_GHOST_SCHED_SKIP_TICK
    - BPF_GHOST_SCHED_PNT
    - BPF_PROG_TYPE_GHOST_MSG
  - BPF helper functions
    - bpf_ghost_wake_agent
    - bpf_ghost_run_gtid;

ghost.run_list  ghost.tasks

task_struct . sched_ghost_entity ghost

ioctl_ghost_commit_txn

  ghost_agent_schedule
    __schedule


scheduler_tick
ghost_tick
tick_handler
ghost_commit_all_greedy_txns
ghost_commit_greedy_txn
    commit_greedy_txn  ==>
      _commit_greedy_txn
        _ghost_commit_pending_txn

            _ghost_commit_txn
              find_task_by_gtid
                find_task_by_pid_ns
                  find_pid_ns

              ghost_move_task
                move_queued_task

              ghost_can_schedule  

              ghost_set_pnt_state
                  ghost_latched_task_preempted
                rq->ghost.latched_task = next

              schedule_next
                resched_curr

              invalidate_cached_tasks    
            ghost_set_txn_state

    // ghost_commit_all_greedy_txns        
    ghost_send_reschedule        





## BPF
__ghost_run_gtid_on
  find_task_by_gtid

  ghost_move_task

  ghost_set_pnt_state

## ghost_run
ghost_run

  schedule_agent ==> done
     schedule_next(GHOST_AGENT_GTID) ==> resched_curr
  ghost_set_pnt_state
  schedule


__ghost_wake_agent_on
  WRITE_ONCE(dst_rq->ghost.agent_should_wake, true)

  set_tsk_need_resched(dst_rq->curr)
  set_preempt_need_resched()



release_from_ghost
    __enclave_remove_task

    submit_enclave_work
      TOBESTUDIED

    ghost_wake_agent_of


__produce_for_task
  ghost_bpf_msg_send

  _produce
    // using ghost_ring


## _ghost_setscheduler
  ghost_prep_agent
    // clean up old txns
    ghost_claim_and_kill_txn

    __ghost_prep_task
      p->ghost.dst_q = q

    p->ghost.agent = true
    ghost_sw_set_flag(p->ghost.status_word, GHOST_SW_TASK_IS_AGENT)
    rq->ghost.agent = p


 ## reap_tasks
 INIT_WORK(&e->task_reaper, enclave_reap_tasks)
      enclave_reap_tasks
        for each ghost_enclave->task_list
            sched_setscheduler_nocheck(t, SCHED_NORMAL

        kref_put(&e->kref, enclave_release) ????      

INIT_WORK(&e->enclave_actual_release, enclave_actual_release);              
      enclave_release
          schedule_work(&e->enclave_actual_release)

enclave_actual_release
     free up all the allocated memory


ghost_vm_ops     
