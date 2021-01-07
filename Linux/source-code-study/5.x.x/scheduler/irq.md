## IRQ

### softirq

#### Key data structures
##### `interrupt.h`
```
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};

```

#### `kernel/softirq.c`
```
__init softirq_init
```

```
open_softirq
  softirq_vec[nr].action = action
```

```
do_softirq
  do_softirq_own_stack
    arch dependent x86_32:
      /* build the stack frame on the softirq stack */
      isp = irqstk + sizeof(*irqstk)
      /* Push the previous esp onto the stack */
      *irqstk = current_stack_pointer
      call_on_stack(__do_softirq, isp)  // call_on_stack is the assemble code      
```

```
__do_softirq
  struct softirq_action *h
  h->action(h)
```

```
irq_enter ==> __irq_enter
irq_exit

```

```
raise_softirq ==> local_irq_save; raise_softirq_irqoff
  __raise_softirq_irqoff
    or_softirq_pending(1UL << nr)   // enable the nr softirq
```
