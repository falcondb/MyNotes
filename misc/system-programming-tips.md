## System Level Programming Tips

### Creating a mmap using huge table
* refer to the code at `linux/tools/kvm/util/util.c`

```
// get the hugetable settings
statfs(htlbfs_path, &sfs)

// sanity check sfs.f_type != HUGETLBFS_MAGIC, size < sfs.f_bsize

// create a temp file, mpath
fd = mkstemp(mpath)

// unlink the name from the file, no one can access it
unlink(mpath)

ftruncate(fd, size)

addr = mmap(NULL, size, PROT_RW, MAP_PRIVATE, fd, 0)

close(fd)

// may call madvise(,, MADV_MERGEABLE)
```


### epoll_event
[epoll for Linux programming](https://programmer.ink/think/epoll-for-linux-programming.html)
```
struct epoll_event epoll_event = {.events = EPOLLIN}

epoll_fd = epoll_create(MAX_EVENTS)

epoll_stop_fd = eventfd(0, 0)
epoll_event.data.fd = epoll_stop_fd

epoll_ctl(epoll_fd, EPOLL_CTL_ADD, epoll_stop_fd, &epoll_event)

pthread_create(&thread, NULL, ioeventfd__thread, NULL)
```

```
ioeventfd__thread
for {
  nfds = epoll_wait(epoll_fd, events, IOEVENTFD_MAX_EVENTS, -1);
  for (i = 0; i < nfds; i++) {
    ioevent = events[i].data.ptr;

    read(ioevent->fd, &tmp, sizeof(tmp)
    ioevent->fn(ioevent->fn_kvm, ioevent->fn_ptr);
  }
}

```
