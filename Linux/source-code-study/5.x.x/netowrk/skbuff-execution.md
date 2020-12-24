## SK Buffer

### SKB allocation
#### `dev_alloc_skb`
```
/**
 *	netdev_alloc_skb - allocate an skbuff for rx on a specific device
 *	@dev: network device to receive on
 *	@length: length to allocate
 *
 *	Allocate a new &sk_buff and assign it a usage count of one. The
 *	buffer has unspecified headroom built in. Users should allocate
 *	the headroom they think they need without accounting for the
 *	built in space. The built in space is used for optimisations.
 */
dev_alloc_skb   ==>  netdev_alloc_skb   ==>   __netdev_alloc_skb    // skbuff.c
  data = page_frag_alloc    // page_alloc.c

  __build_skb(data, len)
    skb = kmem_cache_alloc(skbuff_head_cache, GFP_ATOMIC)
    skb->head = data;
    skb->data = data;
    skb_reset_tail_pointer(skb);
    skb->end = skb->tail + size;
```

#### skbuff.c
```
alloc_skb_with_frags
  alloc_skb => __alloc_skb
    struct kmem_cache = (flags & SKB_ALLOC_FCLONE) ? skbuff_fclone_cache : skbuff_head_cache;
    /* Allocate a new &sk_buff. The returned buffer has no headroom and a
    *	tail room of at least size bytes. The object has a reference count of one */
    skb = kmem_cache_alloc_node ==> kmem_cache_alloc ==> slab_alloc
    data = kmalloc_reserve ==> __kmalloc_reserve
      kmalloc_node_track_caller(__GFP_NOMEMALLOC), if not good, kmalloc_node_track_caller
    skb->head = skb->data = data
    skb->end = skb->tail + size  

  alloc_pages
```

### SKB queue management
* `skb_queue_head`
orders a packet at the header of the specified queue and increments the length of the queue

* `skb_queue_tail`
appends the socket buffer skb to the end of the queue and increments its length

* `skb_dequeue`
removes the top packet from the queue and returns a pointer to it. The length of the queue is decremented by one

* `skb_dequeue_tail`
removes the last packet from a queue and returns a pointer to it

* `skb_queue_purge`
empties the queue list

* `skb_insert`
orders the socket buffer newskb in front of the buffer oldskb in the queue

* `skb_append`
places the socket buffer newskb behind oldskb in the queue of oldskb

* `skb_unlink`
removes the specified socket buffer from its queue and decrements the queue length
