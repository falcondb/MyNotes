## BPF general
[BCC to libbpf conversion guide](https://nakryiko.com/posts/bcc-to-libbpf-howto-guide/#setting-up-user-space-parts)
>

[BPF ring buffer at Andrii Nakryiko's Blog](https://nakryiko.com/posts/bpf-ringbuf/)
Available from Linux version 5.8: BPF map --- BPF ring buffer
> It solves memory efficiency and event re-ordering problems of the BPF perf buffer while meeting or beating its performance

> Perfbuf is a collection of per-CPU circular buffers, which allows to efficiently exchange data between kernel and user-space, two major short-comings: inefficient use of memory and event re-ordering.

> BPF ring buffer is a multi-producer, single-consumer (MPSC) queue and can be safely shared across multiple CPUs simultaneously.

>> Memory overhead: Being shared across all CPUs, BPF ringbuf allows using one big common buffer to deal with this

>> Event ordering: solves this problem by emitting events into a shared buffer

>> Avoid data copying to Perfbuf and out of space error by reserving space first and copying the data to user space directly

>> BPF program has to run from NMI context. BPF ringbuf internally uses a very lightweight spin-lock, which means that data reservation might fail, if lock is contended in NMI context.

[bpf-ringbuf-examples-code](https://github.com/anakryiko/bpf-ringbuf-examples/)

[BPF ring buffer at kernel.org](https://www.kernel.org/doc/html/latest/bpf/ringbuf.html)
BPF_MAP_TYPE_RINGBUF
  * Key and value sizes are enforced to be zero. max_entries is used to specify the size of ring buffer and has to be a power of 2 value.
  * variable-length records;
  * if there is no more space left in ring buffer, reservation fails, no blocking;
  * memory-mappable data area for user-space applications for ease of consumption and high performance;
  * epoll notifications for new incoming data;
  * but still the ability to do busy polling for new data to achieve the lowest latency, if necessary.
MAP APIs
  * bpf_ringbuf_output() allows to copy data from one place to a ring buffer, similarly to bpf_perf_event_output()
  * bpf_ringbuf_reserve() ==> bpf_ringbuf_commit()/bpf_ringbuf_discard()

[Ring buffer selftet in kernel code](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/bpf/progs/test_ringbuf_multi.c)
