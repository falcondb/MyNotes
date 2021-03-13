## Signal
### Data structures

```
struct sighand_struct {
	spinlock_t		siglock;
	refcount_t		count;
	wait_queue_head_t	signalfd_wqh;
	struct k_sigaction	action[_NSIG];
};

struct signal_struct {
  wait_queue_head_t	wait_chldexit;	/* for wait4() */
  /* current thread group signal load-balancing target: */
  struct task_struct	*curr_target;

  /* shared signal handling: */
  struct sigpending	shared_pending;

  /* For collecting multiprocess signals during fork */
  struct hlist_head	multiprocess;

  unsigned int		flags;
};

```

### signal.c
```
send_signal
  Figure out if $force is needed
  __send_signal(sig, struct kernel_siginfo *, struct task_struct *, enum pid_type, force)
    prepare_signal    // handling SIGNSTOP & SIGNCONT?

    pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending : &t->pending

    struct sigqueue *q = __sigqueue_alloc
    list_add_tail(&q->list, &pending->list)

    signalfd_notify // fd notifier

    sigaddset(&pending->signal, sig)

    complete_signal

    trace_signal_generate
```
