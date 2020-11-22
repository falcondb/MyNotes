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
tc qdisc add dev $IFC clsact
tc filter add dev $IFC ingress bpf da obj $1 sec $2
tc qdisc del dev $IFC clsact
```

### parse_varlen.c
Handle vlan in vlan, IP tunnel (IP in IP)
TC_ACT_SHOT, TC drops packages at _TCP:80_ and _UDP:9_. to measure the ingress code path as packets gets dropped in _ip_rcv()_

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

User space of set socket opts
```
getsockopt(SO_BINDTODEVICE,...)
getsockopt(SO_MARK,...)
getsockopt(SO_PRIORITY,...)
```
