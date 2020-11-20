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

## wakeup latency
Attach points: kprobe/try_to_wake_up, tracepoint/sched/sched_switch
```
bpf_get_current_comm
bpf_get_stackid


```
