## Net namespace

### Key data structures
```
/*
 * Generic net pointers are to be used by modules to put some private
 * stuff on the struct net without explicit struct net modification
 *
 * The rules are simple:
 * 1. set pernet_operations->id.  After register_pernet_device you
 *    will have the id of your private pointer.
 * 2. set pernet_operations->size to have the code allocate and free
 *    a private structure pointed to from struct net.
 * 3. do not change this pointer while the net is alive;
 * 4. do not try to have any private reference on the net_generic object.
 *
 * After accomplishing all of the above, the private pointer can be
 * accessed with the net_generic() call.
 */
struct net_generic {
	union {
		struct {
			unsigned int len;
			struct rcu_head rcu;
		} s;
		void *ptr[0];
	};
};
```

### initialization
```
__init net_ns_init
  kmem_cache_create( SMP_CACHE_BYTES, SLAB_PANIC|SLAB_ACCOUNT )
  create_singlethread_workqueue   /* Create workqueue for cleanup */
  net_alloc_generic; rcu_assign_pointer(init_net.gen, ng)
  setup_net(&init_net, &init_user_ns)
    ops_init    // see register_pernet_subsys section
      init_net_initialized = true;
  register_pernet_subsys(&net_ns_ops)
    see register_pernet_subsys section
  rtnl_register( rtnl_net_newid )
  rtnl_register( rtnl_net_getid )
```

### register_pernet_subsys
register ops for a namespace
```
register_pernet_subsys
  allocate and initalize struct pernet_operations.id
  __register_pernet_operations
  if !init_net_initialized
    list_add_tail(&ops->list, list);     return
  ops_init    //
    kzalloc(ops->size, GFP_KERNEL)  // see struct net_generic section
    net_assign_generic(net, *ops->id, data)   
      if old_ng->s.len > id   old_ng->ptr[id] = data;		return 0;
      net_alloc_generic
      memcpy(..., (old_ng->s.len - MIN_PERNET_OPS_ID) * sizeof(void *))
      ng->ptr[id] = data;
	    rcu_assign_pointer(net->gen, ng);     
```
