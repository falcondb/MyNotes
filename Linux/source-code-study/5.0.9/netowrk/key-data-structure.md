# Key data structures and function prototypes

### include/linux/skbuff.h ###
* struct sk_buff {}

### net/netfilter/nf_tables_core.h/c ###
* nf table operations

### net/core/dev.c ###
* network device operations

### include/linux/skmsg.h
* sk_psock_progs
```
struct sk_psock_progs {
	struct bpf_prog			*msg_parser;
	struct bpf_prog			*skb_parser;
	struct bpf_prog			*skb_verdict;
};
```
