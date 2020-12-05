## linux/samples/bpf/cookie_uid_helper_example.c
bpf_insn examples of accessing BPF map. A good reference of learning the BPF byte code

Load the BPF byte code and get the fd, bpf_obj_pin pings the fd to a path, call iptables to attach the Byte code from the pinned file


## Host Bandwidth Management hbm.h/c

```
run_bpf_prog
  prog_load
  setup_cgroup_environment
  create_and_get_cgroup
  join_cgroup ==> join_cgroup_from_top // read pid to cgroup.procs
  bpf_map_update_elem
  bpf_prog_attach
  get tx bytes from /sys/class/net/eth0/statistics/tx_bytes
  bpf_map_update_elem // reset cgroup tx limit by checking the latest eth0 tx bytes
```


## RDMA/infiniband linux/samples/bpf/ibumad_kern.c
Attach points: tracepoint/ib_umad/ib_umad_read_recv; tracepoint/ib_umad/ib_umad_read_send; tracepoint/ib_umad/ib_umad_write

## package data length histogram
```
skb->len
```

Add BPF tunnel encapsulation through _ip_ command
```
ip route add 192.168.253.2/32 encap bpf out obj lwt_len_hist_kern.o section len_hist dev $VETH0
```
use _netserver_ and _netperf_ to run network benchmark

## simple TC linux/samples/bpf/parse_simple/varlan/ldabs.c
```
# add the classier and action queuing discipline
tc qdisc add dev $IFC clsact   
# add bpf as direct action classier to ingress queue
tc filter add dev $IFC ingress bpf da obj $1 sec $2
tc qdisc del dev $IFC clsact
```

### parse_varlen.c
Handle vlan in vlan, IP tunnel (IP in IP)
TC_ACT_SHOT, TC drops packages at _TCP:80_ and _UDP:9_. to measure the ingress code path as packets gets dropped in _ip_rcv()_
if the tunnel device is controlled by a BPF, set _external_ to the tunnel device

### parse_ldabs.c
_load_byte -> asm("llvm.bpf.load.byte")_
> llvm builtin functions that eBPF C program may use to emit BPF_LD_ABS and BPF_LD_IND instructions



## simple count IP packages for protocols linux/samples/bpf/sock_example.c
Raw BPF byte code of parsing IP protocol and update a BPF map


## Set mark and priority through BPF linux/samples/bpf/test_cgrp2_sock.c
Raw BPF byte code of setting mark and priority in _bpf_sock_
```
bpf_load_program(BPF_PROG_TYPE_CGROUP_SOCK, prog, ...)
```

Set socket opts in userspace
```
getsockopt(SO_BINDTODEVICE,...)
getsockopt(SO_MARK,...)
getsockopt(SO_PRIORITY,...)
```

## Parse different L2 and L3 protocols linux/samples/bpf/sockex3_kern.c
L2: IP, IPv6, vlan, MPLS
  * keep the package offset where we have parsed to _skb.cb_ and call _bpf_tail_call_
L3: ICMP, TCP, UDP, IPPROTO_GRE, IPPROTO_IPIP, IPPROTO_IPV6
Where is the code adding the BPF program (their fds) to the jmp_table (BPF_MAP_TYPE_PROG_ARRAY) for the BPF tail calls??

## Test BPF maps at spin lock kprobes spintest_kern.c
Test BPF hash map, percpu hash map and stackmap, attach programs to spin lock kprobes.
Refer how to generate same BPF program at different attach points using C macro


## TC L2 redirect tc_l2_redirect*
Populate a tunnel protocol information in `skb` before call `BPF_redirect`.
```
bpf_skb_set_tunnel_key(skb, ...)
bpf_redirect(*ifindex, flag) // flag for ingress (0) or egress (1)
```

## TCP BPF
* linux/samples/bpf/tcp_basertt_kern.c: returns socket RTT 80ns if the first 5.5 bytes IPv6 addresses of the src and dest are same
* linux/samples/bpf/tcp_bufs_kern.c set: _SO_SNDBUF_ (socket send buffer) and _SO_RCVBUF_ length when an active and passive TCP connection is established.
* linux/samples/bpf/tcp_dumpstats_kern.c: An example of _BPF_MAP_TYPE_SK_STORAGE_ (could be thought of getting the value from a BPF map _sk_ as the key). Also a usage of `bpf_sock_ops_cb_flags_set`


## Test Cgroup
* `union bpf_attr.attach_type` should be `BPF_CGROUP_XXX of enum bpf_attach_type`. `union bpf_attr.target_fd` should be the opened Cgroup file's fd.
* `test_cgrp2_sock.c`, BPF raw byte code of setting socket mark and priority
* dummy network interface for simulating network, `modprobe dummy` `ip link add dummynic type dummy`
* Attach a BPF program to setsockopt at _BPF_CGROUP_INET_SOCK_CREATE_ in a _Cgroup_, then the later new socket should come with the socket opts set in the BPF program
* _BPF_MAP_TYPE_CGROUP_ARRAY_ Array map used to store cgroup fds in user-space for later use in BPF programs which call `bpf_skb_under_cgroup()` defined in _filter.c_ to check if _skb_ is associated with the cgroup in the cgroup array at the specified index.
* `bpf_current_task_under_cgroup` checks if the current task is under the cgroup hierachy, whose fd is in the passed in `BPF_MAP_TYPE_CGROUP_ARRAY` map. `bpf_current_task_under_cgroup` defined in `kernel/trace/bpf_trace.c`, which gets the `struct cgroup` from the `BPF_MAP_TYPE_CGROUP_ARRAY` map and call `task_under_cgroup_hierarchy` defined in `linux/cgroup.h`


## Light-weight tunneling by updating Mac in SKB and redirect test_lwt_bpf.c
`bpf_skb_change_head(skb, 14, 0);` to extend headroom in _skb_ for ETH header (6 + 6 + 2 bytes). `bpf_skb_store_bytes(skb, 0, ...)` to store the ETH header. `bpf_redirect()` if needed

[RFC: Multiprotocol Label Switching Architecture](https://tools.ietf.org/html/rfc3031)
[BPF for lightweight tunnel encapsulation](https://lwn.net/Articles/705609/)

`ip route add $SUBNET encap bpf headroom 14 ...` addes BPF to extend the ETH header. See [ip route man page](https://man7.org/linux/man-pages/man8/ip-route.8.html) for more.
