## Attach BPF program to net filter

### User space
bpf_load_program
socket
bind
setsockopt(sockfd, bpffd)
### Kernel space

```
SYSCALL_DEFINE2(socketcall, ...)
  __sys_setsockopt
    sock_setsockopt || sock->ops->setsockopt
      case SO_ATTACH_BPF
        sk_attach_bpf  ==> __sk_attach_prog
          fp = kmalloc(sizeof(struct sk_filter *), GFP_KERNEL)
          fp->prog = prog
          rcu_assign_pointer(sk->sk_filter, fp)
```
