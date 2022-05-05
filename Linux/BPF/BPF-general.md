## BPF basic
### [BCC to libbpf conversion guide](https://nakryiko.com/posts/bcc-to-libbpf-howto-guide)
- Building everything
  - generating `vmlinux.h` header file with all kernel types
  - compiling your BPF program source code using recent Clang into .o object file
  - generating BPF skeleton header file from compiled BPF object file
  - including generated BPF skeleton header to use from user-space code
  - compiling user-space code, which will get BPF object code embedded in it

- BPF skeleton and BPF app lifecycle
  - BPF maps and global variables, shared between all BPF programs. BPF maps and global variables are also accessible from user-space.
  - goes through the following phases:
    - Open phase `<name>__open()`: BPF object file is parsed, maps, programs, and global variables are discovered, but not created. Possible to make any additional adjustments.
    - Load phase `<name>__load()`: maps are created, various relocations are resolved, BPF programs are loaded into the kernel and verified. No BPF program is yet executed, possible to set up initial BPF map state without racing with the BPF program code execution.
    - Attachment phase `<name>__attach()`: Get attached.
    - Tear down phase `<name>__destroy()`: detached and unloaded, resources are freed.

  - Header includes
    - `vmlinux.h` and few libbpf helper headers

  - Field accesses
    - `BPF_CORE_READ` [BPF CO-RE reference guide](https://nakryiko.com/posts/bpf-core-reference-guide/#bpf-core-read-1)

  - BPF programs
    - `tp/<category>/<name>` for tracepoints
    - `kprobe/<func_name>` for kprobe and `kretprobe/<func_name>` for kretprobe
    - `raw_tp/<name>` for raw tracepoint
    - `cgroup_skb/ingress`, `cgroup_skb/egress`,`cgroup/<subtype>`

  - `#if`
    - `bpf_core_field_exists` checks whether the field exists in a target kernel, returns 1 if exists, 0 otherwise. For cases where there is no readily available variable of desired type, you can write equivalent check, e.g., `(union bpf_attr *)0)->link_create.perf_event.bpf_cookie`, the NULL pointer won't be read, the only purpose is passing the struct layout to the compiler.
    - `bpf_core_type_exists()`
    - `bpf_core_enum_value_exists()`

  - Application configuration
    - Global variables allow user-space control app to pre-setup necessary parameters and flags before a BPF program is loaded and verified.  Either mutable or constant. Mutables for bi-directional exchange of data.
    - BPF code side, `const volatile` for read-only global variables (compiler optimizer mistake), without it defines mutable variables.
    - Variables have to be initialized

  - Global variables
    - BPF code side, we can take global variables' address and pass around into helper functions.
    - User-space side, can be read and updated only through BPF skeleton.
      - `skel->rodata`, `skel->bss`, `skel->data`


### [BPF CO-RE reference guide](https://nakryiko.com/posts/bpf-core-reference-guide/#bpf-core-read-1)

- *CO-RE-relocatable* means that regardless of the actual memory layout of a struct, BPF program will be adjusted to read the field at the correct actual offset relative to the start of a struct.

- `bpf_core_read(dst, sz, src)` like `bpf_probe_read_kernel()`
- the `size` of the field is not automatically relocated, only its offset
- Prefer reading just the primitive fields you are ultimately interested in, not the whole struct
- `bpf_core_read_str()` like `bpf_probe_read_kernel_str()`
- To make such multi-step reads easier to write, libbpf provides the `BPF_CORE_READ()` macro.
- `BPF_CORE_READ_INTO()` macro reads the value of a final field into a destination memory
- `BPF_CORE_READ_STR_INTO()`

- Some BPF program types are "BTF-enabled", which means that BPF verifier in the kernel knows type information associated with input arguments passed into a BPF program. Among such BTF-enabled BPF program types are:
  - BTF-enabled raw tracepoint
  - `fentry/fexit/fmod_ret` BPF programs
  - BPF LSM programs

- `BPF_CORE_READ_BITFIELD()` and `BPF_CORE_READ_BITFIELD_PROBED()`. `_PROBED` variant has to be used when the data to be read has to be *probe read*. They return the value of a bitfield as an u64 integer.
- Whatever the actual nature of a field (any size up to 8 bytes), macros return properly signed-extended 8-byte integers back. `BPF_CORE_READ_BITFIELD()` macros is a universal way to read any integer field regardless of its nature or size.

- `bpf_core_type_size() bpf_core_field_size()` return the size of a field or type in bytes.

- LINUX_KERNEL_VERSION
  - `extern int LINUX_KERNEL_VERSION __kconfig `, e.g., `if (LINUX_KERNEL_VERSION > KERNEL_VERSION(5, 15, 0))`

- Kconfig extern variables
  - through `/proc/config.gz`
  - example: `extern bool CONFIG_BPF_JIT_ALWAYS_ON __kconfig __weak`

- Relocatable enums: `bpf_core_enum_value()`

- For any type, field, enum, or enumerator, if the entity's name contains a suffix of the form `___something` , such name suffix is ignored for the purposes of CO-RE relocation as if it was never there.  `struct task_struct___my_own_copy` is equivalent to `struct task_struct` in BPF application

- Reading kernel data structures from user-space memory
  - `bpf_core_read_user() bpf_core_read_user_str() BPF_CORE_READ_USER_STR_INTO() BPF_CORE_READ_USER_INTO() BPF_CORE_READ_USER()` through `bpf_probe_read_user`

- Capturing BTF type IDs
  - *TO BE STUDIED AGAIN*
  - `bpf_core_type_id_kernel() bpf_core_type_id_local() `


### [BBC libbpf tools](https://github.com/iovisor/bcc/tree/master/libbpf-tools)


### [Add code-generated BPF object skeleton support](https://lwn.net/Articles/806911/)
  - thanks to BPF array `mmap()` support, working with global data (variables) from userspace is now as natural as it is from BPF side.
  - use it to pre-populate BPF maps at creation time, and will re-mmap() BPF map's contents at exactly the same userspace memory address such that it can continue working with all the same pointers without any interruptions. If kernel doesn't support mmap(), global data will still be successfully initialized, but after map creation global data structures inside skeleton will be NULL-ed out. This allows userspace application to gracefully handle lack of mmap() support, if necessary. **HAVE TO READ THE CODE TO UNDERSTAND IT**


## BPF advanced

### [BPF CO-RE (Compile Once – Run Everywhere)](https://nakryiko.com/posts/bpf-portability-and-co-re/)
   - High-level BPF CO-RE mechanics
    - BPF CO-RE brings together necessary pieces of functionality and data at all levels of the software stack: kernel, user-space BPF loader library (libbpf), and compiler (Clang) – to make it possible and easy to write BPF programs in a portable manner, handling discrepancies between different kernels within the same pre-compiled BPF program.

  - *BTF (BPF Type Format)* type information, which allows to capture crucial pieces of information about kernel and BPF program types and code, enabling all the other parts of BPF CO-RE puzzle
  - compiler (Clang) provides means for BPF program C code to express the intent and record relocation information
  - BPF loader (libbpf) ties BTFs from kernel and BPF program together to adjust compiled BPF code to specific kernel on target hosts
  - kernel, while staying completely BPF CO-RE-agnostic, provides advanced BPF features to enable some of the more advanced scenarios  

  - created as an alternative to a more generic and verbose DWARF debug information. BTF is a space-efficient, compact, yet still expressive enough format

  - `/sys/kernel/btf/vmlinux`
  - `bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h` to get the `vmlinux.h`
  - Most commonly missing ones might be provided as part of libbpf’s `bpf_helpers.h`

### [BPF Portability and CO-RE](https://facebookmicrosites.github.io/bpf/blog/2020/02/19/bpf-portability-and-co-re.html)

### [BTF deduplication and Linux kernel BTF](https://nakryiko.com/posts/btf-dedup/)
-  BTF represents each type with one of a few possible type descriptors identified by a kind: `BTF_KIND_INT, BTF_KIND_ENUM, BTF_KIND_STRUCT, BTF_KIND_UNION, BTF_KIND_ARRAY, BTF_KIND_FWD, BTF_KIND_PTR, BTF_KIND_CONST, BTF_KIND_VOLATILE, BTF_KIND_RESTRICT, BTF_KIND_TYPEDEF`

-  `pahole` was recently updated with the ability to convert DWARF into corresponding BTF type information in a straightforward, one-to-one fashion.

- the deduplication algorithm, which removes and compacts the type information in DWARF format collected from different compilation units.
  - Strings deduplication: compacts strings into a char array with `\0` separator, and strings are identified by the outset
  - non-reference types deduplication:
### [Building BPF applications with libbpf-bootstrap](https://nakryiko.com/posts/libbpf-bootstrap/)



### [Introduce BPF trampoline](https://lwn.net/Articles/804112/)
  - BPF trampoline that works as a bridge between kernel functions, BPF programs and other BPF programs
  -  `fentry/fexit` BPF programs that are roughly equivalent to `kprobe/kretprobe`. practically zero overhead.

  - BPF trampoline allows attaching similar fentry/fexit BPF program to any networking BPF program.

  - The BPF trampoline will be used to dynamicly link BPF programs.


### [BPF Type Format (BTF) @kernel doc](https://www.kernel.org/doc/html/latest/bpf/btf.html)


### [BPF ring buffer at Andrii Nakryiko's Blog](https://nakryiko.com/posts/bpf-ringbuf/)
Available from Linux version 5.8: BPF map --- BPF ring buffer
> It solves memory efficiency and event re-ordering problems of the BPF perf buffer while meeting or beating its performance

> Perfbuf is a collection of per-CPU circular buffers, which allows to efficiently exchange data between kernel and user-space, two major short-comings: inefficient use of memory and event re-ordering.

> BPF ring buffer is a multi-producer, single-consumer (MPSC) queue and can be safely shared across multiple CPUs simultaneously.

>> Memory overhead: Being shared across all CPUs, BPF ringbuf allows using one big common buffer to deal with this

>> Event ordering: solves this problem by emitting events into a shared buffer

>> Avoid data copying to Perfbuf and out of space error by reserving space first and copying the data to user space directly

>> BPF program has to run from NMI context. BPF ringbuf internally uses a very lightweight spin-lock, which means that data reservation might fail, if lock is contended in NMI context.

### [bpf-ringbuf-examples-code](https://github.com/anakryiko/bpf-ringbuf-examples/)

### [BPF ring buffer at kernel.org](https://www.kernel.org/doc/html/latest/bpf/ringbuf.html)
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

### [Ring buffer selftet in kernel code](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/bpf/progs/test_ringbuf_multi.c)
