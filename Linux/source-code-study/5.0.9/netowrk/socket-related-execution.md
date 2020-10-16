## Socket related execution in Kernel

### net/socket.c

#### syscall root
```
SYSCALL_DEFINE2(socketcall, ...)
  __sys_socket
  __sys_bind
  __sys_connect
  __sys_listen
  __sys_accept4
  __sys_sendto
  __sys_recvfrom
  __sys_setsockopt
  __sys_sendmsg
  __sys_recvmsg

```

#### socket
```
__sys_socket
  sock_create
    __sock_create
      sock_alloc
      net_proto_family.create

  sock_map_fd
    get_unused_fd_flags
    sock_alloc_file
    fd_install

```

#### setsockopt
```
__sys_setsockopt
  sockfd_lookup_light
    file->private_data
    BPF_CGROUP_RUN_PROG_SETSOCKOPT // BPF program attached
    if level == SOL_SOCKET
      sock_setsockopt
    else
       sock->ops->setsockopt
```

#### socket bind / listen / connect
```
__sys_bind
  sockfd_lookup_light
  sock->ops->bind / listen / connect
```


####  send / sendto
```
__sys_sendto
  sockfd_lookup_light
  sock_sendmsg
    sock_sendmsg_nosec
    INDIRECT_CALL_INET(sock->ops->sendmsg, inet6_sendmsg, inet_sendmsg, ...)
      INDIRECT_CALL_2(sk->sk_prot->sendmsg, tcp_sendmsg, udp_sendmsg
      or
      INDIRECT_CALL_2(sk->sk_prot->sendmsg, tcp_sendmsg, udpv6_sendmsg, ...)
```

#### tcp in net/ipv4/tcp
```
tcp_sendmsg_locked
  TODO:
```

```
sock_alloc_file
  alloc_file_pseudo
  sock->file = file;
  file->private_data = sock;
  stream_open(SOCK_INODE(sock), file); //struct socket_alloc { struct socket;	struct inode; }
  };
```
