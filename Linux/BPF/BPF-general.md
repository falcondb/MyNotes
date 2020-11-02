## BPF general

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
[Ring buffer selftet in kernel code](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/bpf/progs/test_ringbuf_multi.c)
