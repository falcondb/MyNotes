## SK Buffer

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
