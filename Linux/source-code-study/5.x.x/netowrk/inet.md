## inet Internet protocol family
The _Internet protocol family_ is a collection of protocols layered atop the Internet Protocol (IP) transport layer, and utilizing the Internet address format. The Internet family provides protocol support for the _SOCK_STREAM_, _SOCK_DGRAM_, and _SOCK_RAW_ socket types; the _SOCK_RAW_ interface provides access to the IP protocol.
A
Internet addresses are four byte quantities, stored in network standard format. Sockets bound to the _Internet protocol family_ utilize the following addressing structure,
```struct sockaddr_in {
	uint8_t		sin_len;
	sa_family_t	sin_family;
	in_port_t	sin_port;
	struct in_addr	sin_addr;
	char		sin_zero[8];
};
```
Sockets may be created with the local address _INADDR_ANY_ to affect “wildcard” matching on incoming messages. The address in a `connect` or `sendto` call may be given as _INADDR_ANY_ to mean “this host”. The distinguished address _INADDR_BROADCAST_ is allowed as a shorthand for the broadcast address on the primary network if the first network configured supports broadcast.

The _Internet protocol family_ is comprised of the IP network protocol, _Internet Control Message Protocol (ICMP)_, _Internet Group Management Protocol (IGMP)_, _Transmission Control Protocol (TCP)_, and _User Datagram Protocol (UDP)_. TCP is used to support the _SOCK_STREAM_ abstraction while UDP is used to support the _SOCK_DGRAM_ abstraction. A raw interface to IP is available by creating an Internet socket of type _SOCK_RAW_. The ICMP message protocol is accessible from a raw socket.


### `net/inet_connection_sock.h`
```
struct inet_connection_sock_af_ops {
	int	    (*queue_xmit)();
	void	    (*send_check)();
	int	    (*rebuild_header)();
	void	    (*sk_rx_dst_set)();
	int	    (*conn_request)();
	struct sock *(*syn_recv_sock)();
	u16	    net_header_len;
	u16	    net_frag_header_len;
	u16	    sockaddr_len;
	int	    (*setsockopt)();
	int	    (*getsockopt)();
	void	    (*addr2sockaddr)();
	void	    (*mtu_reduced)(struct sock *sk);
};


/** inet_connection_sock - INET connection oriented sock
 * @icsk_accept_queue:	   FIFO of established children
 * @icsk_bind_hash:	   Bind node
 * @icsk_timeout:	   Timeout
 * @icsk_retransmit_timer: Resend (no ack)
 * @icsk_rto:		   Retransmit timeout
 * @icsk_pmtu_cookie	   Last pmtu seen by socket
 * @icsk_ca_ops		   Pluggable congestion control hook
 * @icsk_af_ops		   Operations which are AF_INET{4,6} specific
 * @icsk_ulp_ops	   Pluggable ULP control hook
 * @icsk_ulp_data	   ULP private data
 * @icsk_clean_acked	   Clean acked data hook
 * @icsk_listen_portaddr_node	hash to the portaddr listener hashtable
 * @icsk_ca_state:	   Congestion control state
 * @icsk_retransmits:	   Number of unrecovered [RTO] timeouts
 * @icsk_pending:	   Scheduled timer event
 * @icsk_backoff:	   Backoff
 * @icsk_syn_retries:      Number of allowed SYN (or equivalent) retries
 * @icsk_probes_out:	   unanswered 0 window probes
 * @icsk_ext_hdr_len:	   Network protocol overhead (IP/IPv6 options)
 * @icsk_ack:		   Delayed ACK control data
 * @icsk_mtup;		   MTU probing control data
 */
struct inet_connection_sock {
	struct inet_sock	  icsk_inet;
	struct request_sock_queue icsk_accept_queue;
	struct inet_bind_bucket	  *icsk_bind_hash;
	unsigned long		  icsk_timeout;
 	struct timer_list	  icsk_retransmit_timer;
 	struct timer_list	  icsk_delack_timer;
	__u32			  icsk_rto;
	__u32			  icsk_pmtu_cookie;
	const struct tcp_congestion_ops *icsk_ca_ops;
	const struct inet_connection_sock_af_ops *icsk_af_ops;
	const struct tcp_ulp_ops  *icsk_ulp_ops;
	void __rcu		  *icsk_ulp_data;
	void (*icsk_clean_acked)(struct sock *sk, u32 acked_seq);
	struct hlist_node         icsk_listen_portaddr_node;
	unsigned int		  (*icsk_sync_mss)(struct sock *sk, u32 pmtu);
	__u8			  icsk_ca_state:6,
				  icsk_ca_setsockopt:1,
				  icsk_ca_dst_locked:1;
	__u8			  icsk_retransmits;
	__u8			  icsk_pending;
	__u8			  icsk_backoff;
	__u8			  icsk_syn_retries;
	__u8			  icsk_probes_out;
	__u16			  icsk_ext_hdr_len;
	struct {
		__u8		  pending;	 /* ACK is pending			   */
		__u8		  quick;	 /* Scheduled number of quick acks	   */
		__u8		  pingpong;	 /* The session is interactive		   */
		__u8		  blocked;	 /* Delayed ACK was blocked by socket lock */
		__u32		  ato;		 /* Predicted tick of soft clock	   */
		unsigned long	  timeout;	 /* Currently scheduled timeout		   */
		__u32		  lrcvtime;	 /* timestamp of last received data packet */
		__u16		  last_seg_size; /* Size of last incoming segment	   */
		__u16		  rcv_mss;	 /* MSS used for delayed ACK decisions	   */
	} icsk_ack;
	struct {
		int		  enabled;
		/* Range of MTUs to search */
		int		  search_high;
		int		  search_low;
		/* Information on the current probe. */
		int		  probe_size;
		u32		  probe_timestamp;
	} icsk_mtup;
	u32			  icsk_user_timeout;
	u64			  icsk_ca_priv[104 / sizeof(u64)];
#define ICSK_CA_PRIV_SIZE      (13 * sizeof(u64))
};
```

### `ipv4/af_inet.c`
```
static const struct net_proto_family inet_family_ops = {
	.family = PF_INET,  //#define PF_INET		AF_INET   2
	.create = inet_create,
	.owner	= THIS_MODULE,
};

__init inet_init
  proto_register(&tcp_prot / &udp_prot / &raw_prot /&ping_prot , 1);
  sock_register
    rcu_assign_pointer(net_families[ops->family], ops);  //socket.c
  inet_add_protocol(&icmp_protocol / &udp_protocol / &tcp_protocol / &igmp_protocol
    cmpxchg(&inet_protos[protocol], NULL, prot)   // inet_protos defined in ipv4/protocol.c
  arp_init
  ip_init
  tcp_init
    tcp_v4_init
      register_pernet_subsys(&tcp_sk_ops)
    tcp_tasklet_init        // net/ipv4/tcp_output.c
    for_each_possible_cpu(i)
      struct tsq_tasklet *tsq = &per_cpu(tsq_tasklet, i)
        tasklet_init(&tsq->tasklet, v, tsq)
  udp_init
  raw_init
  ping_init
  icmp_init
  ipv4_proc_init    // proc fs init
    raw_proc_init tcp4_proc_init udp4_proc_init
     register_pernet_subsys(&tcp4_net_ops)
  ipfrag_init
  dev_add_pack    // add packet handler

  ip_tunnel_core_init

```

### INET device management
#### `inetdevice.h`
```
struct in_device {
	struct net_device	*dev;
	refcount_t		refcnt;
	int			dead;
	struct in_ifaddr	__rcu *ifa_list;/* IP ifaddr chain		*/
	struct ip_mc_list __rcu	*mc_list;	/* IP multicast filter chain    */
	struct ip_mc_list __rcu	* __rcu *mc_hash;
	int			mc_count;	/* Number of installed mcasts	*/
	spinlock_t		mc_tomb_lock;
	struct ip_mc_list	*mc_tomb;
	unsigned long		mr_v1_seen;
	unsigned long		mr_v2_seen;
	unsigned long		mr_maxdelay;
	unsigned long		mr_qi;		/* Query Interval */
	unsigned long		mr_qri;		/* Query Response Interval */
	unsigned char		mr_qrv;		/* Query Robustness Variable */
	unsigned char		mr_gq_running;
	unsigned char		mr_ifc_count;
	struct timer_list	mr_gq_timer;	/* general query timer */
	struct timer_list	mr_ifc_timer;	/* interface change timer */
	struct neigh_parms	*arp_parms;
	struct ipv4_devconf	cnf;
	struct rcu_head		rcu_head;
};
```
An in_device structure is created for each network device that was configured for the Internet Protocol
```
struct in_ifaddr {
	struct hlist_node	hash;
	struct in_ifaddr	__rcu *ifa_next;
	struct in_device	*ifa_dev;
	struct rcu_head		rcu_head;
	__be32			ifa_local;
	__be32			ifa_address;
	__be32			ifa_mask;
	__u32			ifa_rt_priority;
	__be32			ifa_broadcast;
	unsigned char		ifa_scope;
	unsigned char		ifa_prefixlen;
	__u32			ifa_flags;
	char			ifa_label[IFNAMSIZ];
	/* In seconds, relative to tstamp. Expiry is at tstamp + HZ * lft. */
	__u32			ifa_valid_lft;
	__u32			ifa_preferred_lft;
	unsigned long		ifa_cstamp; /* created timestamp */
	unsigned long		ifa_tstamp; /* updated timestamp */
};
```

#### `net/ipv4/devinet.c`

```
__init devinet_init
```
