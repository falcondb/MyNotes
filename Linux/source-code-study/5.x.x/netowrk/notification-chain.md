## Notification Chain

### Key data structues

* `notifier.h`
```
struct notifier_block {
	notifier_fn_t notifier_call;
	struct notifier_block __rcu *next;
	int priority;
};

typedef	int (*notifier_fn_t)(struct notifier_block *nb, unsigned long action, void *data);

```

* `ipv4/devinet.c`
```
BLOCKING_NOTIFIER_HEAD(inetaddr_chain)    // notifier.h
```


* `net/core/dev.c`
```
RAW_NOTIFIER_HEAD(netdev_chain)       // notifier.h
```

### Key functions

```
struct notifier_block **nl, struct notifier_block *n
  // add n to the nl list where n->priority > (*nl)->priority or to the tail
```

```
notifier_call_chain
  // make the nb->notifier_call in the list
  // stop at the pass-in number of calls, or  NOTIFY_STOP_MASK
```
