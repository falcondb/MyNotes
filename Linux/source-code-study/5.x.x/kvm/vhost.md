## Vhost

### Userspace usage

#### Creating a Vhost

```
vhost_fd = open("/dev/vhost-net", O_RDWR)

// make an instance of struct vhost_memory with one or multi struct vhost_memory_region
struct vhost_memory *mem
mem = calloc(1, sizeof(*mem) + sizeof(struct vhost_memory_region))

ioctl(vhost_fd, VHOST_SET_OWNER)
ioctl(vhost_fd, VHOST_SET_MEM_TABLE, mem)

free(mem)
```

#### Notifying a Vhost
```
struct vhost_vring_file file = {
  .index	= vq,
  .fd	= efd,
}

ioctl(ndev->vhost_fd, VHOST_SET_VRING_KICK, &file)
```

#### [Vhost code example at DPDK](https://doc.dpdk.org/api/examples_2vhost_blk_2vhost_blk_8c-example.html)

#### [Examples in Linux source code](https://elixir.bootlin.com/linux/v5.2.21/source/tools/virtio/virtio_test.c)

```
vdev_info_init
  open("/dev/vhost-test", O_RDWR)
  ioctl(dev->control, VHOST_SET_OWNER, NULL)

  dev->mem->regions[0].guest_phys_addr = XXXX
  dev->mem->regions[0].userspace_addr =  XXXX

  ioctl(dev->control, VHOST_SET_MEM_TABLE, dev->mem)

vq_info_add
  info->kick = eventfd(0, EFD_NONBLOCK)
  info->call = eventfd(0, EFD_NONBLOCK)

  vring_init(&info->vring, ...)  \\ init desc, avail, used in the ring

  info->vq = vring_new_virtqueue
    vring_init(&vq->vring, ..., vq_notify, vq_callback, ...)

  ioctl(dev->control, VHOST_SET_FEATURES, &features)
  ioctl(dev->control, VHOST_SET_VRING_NUM, &state)
  ioctl(dev->control, VHOST_SET_VRING_BASE, &state)
  ioctl(dev->control, VHOST_SET_VRING_ADDR, &addr)
  ioctl(dev->control, VHOST_SET_VRING_KICK, &file)
  ioctl(dev->control, VHOST_SET_VRING_CALL, &file)

vq_notify
    write(info->kick, ...)

vq_callback
    \\ no ops, ignore delivery acknowledge
```

### Kernel implementation

```
* OK, now we need to know about added descriptors. */
vhost_enable_notify

```

```
vhost_add_used_and_signal // similar to vhost_add_used_and_signal_n
  vhost_add_used
    struct vring_used_elem heads = { head, len }
    /* After we've used one of their buffers, we tell them about it.  We'll then want to notify the guest, using eventfd. */
    vhost_add_used_n(vq, &heads, 1)
      start = vq->last_used_idx % vq->num
      n = vq->num - start
      if n < count
        __vhost_add_used_n(vq, heads, n)
        heads += n  // head is a pointerof vring_used_elem
        count -= n
      __vhost_add_used_n(vq, heads, count)  
        start = vq->last_used_idx % vq->num
        used = vq->used->ring + start
        // copy the index and len from struct vring_used_elem *heads to used by __put_user or __copy_to_user
      put_user(vq->last_used_idx, &vq->used->idx)
      vq->last_used_idx += count

  vhost_signal
    eventfd_signal
```


```
/* This looks in the virtqueue and for the first available buffer, and converts
 * it to an iovec for convenient access.  Since descriptors consist of some
 * number of output then some number of input descriptors, it's actually two
 * iovecs, but we pack them into one and note how many of each there were.
 */
vhost_get_vq_desc
  __get_user(head, &vq->avail->ring[last_avail_idx % vq->num]))

  i = head
  do {
    __copy_from_user(&desc, vq->desc + i, sizeof desc)

    if VRING_DESC_F_INDIRECT  // a list of buffer descriptors
      get_indirect

    // translate the struct vring_desc to struct iovec[]
    ret = translate_desc
      
    if VRING_DESC_F_WRITE
      *in_num += ret
    else
      *out_num += ret  

  } while ((i = next_desc(&desc)) != -1)

  vq->last_avail_idx++
```
