## Linux data structures

### Linked list
[How does the kernel implements Linked Lists?](https://kernelnewbies.org/FAQ/LinkedLists)
When first encountering this, most people are confused because they have been taught to implement linked lists by adding a pointer in a structure which points to the next similar structure in the linked list. The drawback of this approach, and the reason for which the kernel implements linked lists differently, is that you need to write code to handle adding / removing / etc elements specifically for that data structure. Here, we can add a `struct list_head` field to any other data structure and, as we'll see shortly, make it a part of a linked list.

* initialization
`linux/types.h`
```
struct list_head {
	struct list_head *next, *prev;
};

struct hlist_head {
	struct hlist_node *first;
};

struct hlist_node {
	struct hlist_node *next, **pprev;
};
```

```
struct mystruct {
  int data ;
  struct list_head mylist ;
};

struct mystruct first = {
     .data = 10,
     .mylist = LIST_HEAD_INIT(first.mylist)
} ;

#define LIST_HEAD_INIT(name) { &(name), &(name) }

struct mystruct second ;
second.data = 20 ;
INIT_LIST_HEAD( & second.mylist ) ;

LIST_HEAD(mylinkedlist)
#define LIST_HEAD(name) \
        struct list_head name = LIST_HEAD_INIT(name)      
```
* adding nodes
```
list_add ( &first.mylist , &mylinkedlist ) ;

static inline void list_add(struct list_head *new, struct list_head *head)
{
      __list_add(new, head, head->next);
}
static inline void __list_add(struct list_head *new, struct list_head *prev, struct list_head *next)
{
  next->prev = new;
  new->next = next;
  new->prev = prev;
  prev->next = new;
}    
```

* iteration
```
#define list_for_each(pos, head) \
    for (pos = (head)->next; pos != (head); pos = pos->next)
```

```
list_for_each ( position , & mylinkedlist )
{
     datastructureptr = list_entry ( position, struct mystruct , mylist );
     printk ("data  =  %d\n" , datastructureptr->data );
}
```

```
#define list_for_each_entry(pos, head, member)                          \
  for (pos = list_entry((head)->next, typeof(*pos), member);      \
             &pos->member != (head);        \
             pos = list_entry(pos->member.next, typeof(*pos), member))

list_for_each_entry ( datastructureptr , & mylinkedlist, mylist )
{
     printk ("data  =  %d\n" , datastructureptr->data );
}
```

### Hash table
[How does the kernel implements Hashtables?](https://kernelnewbies.org/FAQ/Hashtables)

* initialization
```
#define DEFINE_HASHTABLE(name, bits)                                            \
        struct hlist_head name[1 << (bits)] =                                   \
                         { [0 ... ((1 << (bits)) - 1)] = HLIST_HEAD_INIT }

DEFINE_HASHTABLE(a, 3)                         

```

* adding nodes
```
#define hash_add_rcu(hashtable, node, key)                                      \
       hlist_add_head_rcu(node, &hashtable[hash_min(key, HASH_BITS(hashtable))])


struct mystruct {
    int data ;
    struct hlist_node next ;
}
struct mystruct first = {
    .data = 10,
    .next = 0    /* Will be initilaized when added to the hashtable */
}

hash_add(a, &first.next, first.data)
```


* iteration
```
#define hash_for_each(name, bkt, obj, member)                           \
     for ((bkt) = 0, obj = NULL; obj == NULL && (bkt) < HASH_SIZE(name);\
                        (bkt)++)\
                    hlist_for_each_entry(obj, &name[bkt], member)
```


### container_of
[container_of](https://kernelnewbies.org/MagicMacros)
[linux kernel monkey log - Greg Kroah-Hartman](http://www.kroah.com/log/linux/container_of.html)
```
#define container_of(ptr, type, member) ({                      \
        const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
        (type *)( (char *)__mptr - offsetof(type,member) );})
```
