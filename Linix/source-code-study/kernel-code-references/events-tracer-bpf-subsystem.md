# events, trace, BPF & Perf #
[Ftrace, perf, and the tracing ABI](https://lwn.net/Articles/442113/)


## Tracer ##

## Trace points ##
[Using the TRACE_EVENT() macro part 1](https://lwn.net/Articles/379903/)

The post introduces the basic steps to create a trace point, using TRACE_EVENT() macro

[Using the TRACE_EVENT() macro part 2](https://lwn.net/Articles/381064/)

The post introduces the DECLARE_EVENT_CLASS() macro to reduce the code/data size of the trace points

[Using the TRACE_EVENT() macro part 3](https://lwn.net/Articles/383362/)

This is the fun part and the magic of TRACE_EVENT(), it is fun to read it more.


## Kprobe ##
[An introduction to KProbes](https://lwn.net/Articles/132196/)

Introduction of the interface and internal of IBM Kprobes

## eBPF ##
[A thorough introduction to eBPF](https://lwn.net/Articles/740157/)

A brief history of eBPF and its key components, more like eBPF-101

[tracing: accelerate tracing filters with BPF](https://lwn.net/Articles/598545/)

TO REEEEEAAAAAAAD

[Tracepoints with BPF](https://lwn.net/Articles/683504/)

READ this post, if really want to hack BPF!!!
It explain how BPF link to events and tracefs, and the argument format of BPF call!
[tplist](https://github.com/iovisor/bcc/blob/master/tools/tplist.py), a tool of converting tracepoint format to C struct


[Linux Socket Filtering aka Berkeley Packet Filter](https://www.kernel.org/doc/Documentation/networking/filter.txt)

I haven't read it through, seems to me a lot of details. Will revisit it when more about eBPF is understood.

[BPF: the universal in-kernel virtual machine](https://lwn.net/Articles/599755/)


[A JIT for packet filters](https://lwn.net/Articles/437981/)

A description of JIT for eBPF, code maybe more understandable, in the arch/

[Yet another new approach to seccomp](https://lwn.net/Articles/475043/)


## Perf ##
[Raw events and the perf ABI](https://lwn.net/Articles/441209/)

