## Perf + BPF

### Definitions
* uapi/linux/perf_event.h
* [Linux Man page](https://man7.org/linux/man-pages/man2/perf_event_open.2.html)
```
struct perf_event_attr {
	__u32			type;
	__u32			size;
	__u64			config;
	union {
		__u64		sample_period;
		__u64		sample_freq;
	};
  ...

  __u32			bp_type;
	union {
		__u64		bp_addr;
		__u64		kprobe_func; /* for perf_kprobe */
		__u64		uprobe_path; /* for perf_uprobe */
		__u64		config1; /* extension of config */
	};
	union {
		__u64		bp_len;
		__u64		kprobe_addr; /* when kprobe_func == NULL */
		__u64		probe_offset; /* for perf_[k,u]probe */
		__u64		config2; /* extension of config1 */
	};
  union {
		__u32		wakeup_events;	  /* wakeup every n events */
		__u32		wakeup_watermark; /* bytes before wakeup   */
	};
  	__u64	branch_sample_type;
  ...
}


  /*
   * Structure used by below PERF_EVENT_IOC_QUERY_BPF command
   * to query bpf programs attached to the same perf tracepoint
   * as the given perf event.
   */
  struct perf_event_query_bpf {
  	__u32	ids_len;
  	__u32	prog_cnt;
  	__u32	ids[0];
  };


  /*
   * Structure of the page that can be mapped via mmap
   * The mmap size should be 1+2^n pages, where the first page is a meta‐
   * data page (struct perf_event_mmap_page) that contains various bits of
   * information such as where the ring-buffer head is.
   */
  struct perf_event_mmap_page {
  	__u32	version;		/* version number of this structure */
  	__u32	compat_version;		/* lowest version this is compat with */
  	__u32	lock;			/* seqlock for synchronization */
  	__u32	index;			/* hardware event identifier */
  	__s64	offset;			/* add to hardware event value */
  	__u64	time_enabled;		/* time event active */
  	__u64	time_running;		/* time event on cpu */
  	union {
  		__u64	capabilities;
  		struct {
  			__u64	cap_bit0		: 1, /* Always 0, deprecated, see commit 860f085b74e9 */
  				cap_bit0_is_deprecated	: 1, /* Always 1, signals that bit 0 is zero */

  				cap_user_rdpmc		: 1, /* The RDPMC instruction can be used to read counts */
  				cap_user_time		: 1, /* The time_* fields are used */
  				cap_user_time_zero	: 1, /* The time_zero field is used */
  				cap_____res		: 59;
  		};
  	};
  	__u16	pmc_width;
  	__u16	time_shift;
  	__u32	time_mult;
  	__u64	time_offset;
  	__u64	time_zero;
  	__u32	size;		
  	__u8	__reserved[118*8+4];	/* align to 1k. */

  	/*
  	 * Control data for the mmap() data buffer.
  	 */
  	__u64   data_head;		/* head in the data section */
  	__u64	data_tail;		/* user-space written tail */
  	__u64	data_offset;		/* where the buffer starts */
  	__u64	data_size;		/* data buffer size */
  	__u64	aux_head;
  	__u64	aux_tail;
  	__u64	aux_offset;
  	__u64	aux_size;
  };


  struct perf_event_header {
	__u32	type;
	__u16	misc;
	__u16	size;
};


struct perf_sample_data {
	u64				addr;
	struct perf_raw_record		*raw;
	struct perf_branch_stack	*br_stack;
	u64				period;
	u64				weight;
	u64				txn;
	union  perf_mem_data_src	data_src;
	u64				type;
	u64				ip;
	struct {
		u32	pid;
		u32	tid;
	}				tid_entry;
	u64				time;
	u64				id;
	u64				stream_id;
	struct {
		u32	cpu;
		u32	reserved;
	}				cpu_entry;
	struct perf_callchain_entry	*callchain;
	u64				aux_size;

	struct perf_regs		regs_user;
	struct pt_regs			regs_user_copy;

	struct perf_regs		regs_intr;
	u64				stack_user_size;

	u64				phys_addr;
} ____cacheline_aligned;

```


`enum perf_event_type`
* Each type has its own ABI with struct perf_event_header and its private data.
* Refer to [Linux Man page](https://man7.org/linux/man-pages/man2/perf_event_open.2.html) for the details

### BPF output through Perf
```
struct perf_event_attr attr = {
  .sample_type	= PERF_SAMPLE_RAW,
  .type		= PERF_TYPE_SOFTWARE,
  .config		= PERF_COUNT_SW_BPF_OUTPUT,
  .wakeup_events	= 1,
};
```


* bpf_perf_event_read_simple in `tools/lib/bpf/libbpf.c` for user space
  * set `struct perf_event_mmap_page *` to the mmap address
  * get the ring buffer head and tail after the 1st page for struct perf_event_mmap_page
  * get `struct perf_event_header` from the current tail
  * get the data appended to the `struct perf_event_header` if the package across the ring
  * call the pass-in function with the header
  * notes: read the ring buffer head and tail by `smp_load_acquire` & `smp_store_release`

* bpf_perf_event_output `kernel/trace/bpf_trace.c` for kernel space
  * __bpf_perf_event_output
    * get `struct bpf_array` from the map
    * get `struct bpf_event_entry` from the `struct bpf_array`, then `struct perf_event`
    * perf_event_output ==> __perf_event_output
      * perf_prepare_sample
        * for PERF_SAMPLE_TID, PERF_SAMPLE_TIME, PERF_SAMPLE_IDENTIFIER, PERF_SAMPLE_STREAM_ID, PERF_SAMPLE_CPU, __perf_event_header__init_id
        * for PERF_SAMPLE_CALLCHAIN, perf_callchain.
        * for PERF_SAMPLE_REGS_USER | PERF_SAMPLE_STACK_USER, perf_sample_regs_user
        * for PERF_SAMPLE_REGS_INTR, perf_sample_regs_intr
        * for PERF_SAMPLE_PHYS_ADDR, perf_virt_to_phys : virt_to_phys for kernel address, __get_user_pages_fast and page_to_phys for user address
        * for others, calculate the size for the `struct perf_event_header`
      * output_begin
        * __perf_output_begin in kernel/evetns/ring_buffer.c, handles the ring buffer, sample loss.
      * perf_output_sample
        * perf_output_put ==> perf_output_copy ==> output_copy different types of data
        * PERF_SAMPLE_RAW, perf_output_put
        * perf_output_put with data in `struct perf_sample_data`
      * perf_output_end
        * handle the head with memory sync ops
