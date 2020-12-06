## SK Buffer key data structures

### skbuff.h

```
/**
 *	struct sk_buff - socket buffer
 *	@next: Next buffer in list
 *	@prev: Previous buffer in list
 *	@tstamp: Time we arrived/left
 *	@rbnode: RB tree node, alternative to next/prev for netem/tcp
 *	@sk: Socket we are owned by
 *	@dev: Device we arrived on/are leaving by
 *	@cb: Control buffer. Free for use by every layer. Put private vars here
 *	@_skb_refdst: destination entry (with norefcount bit)
 *	@sp: the security path, used for xfrm
 *	@len: Length of actual data
 *	@data_len: Data length
 *	@mac_len: Length of link layer header
 *	@hdr_len: writable header length of cloned skb
 *	@csum: Checksum (must include start/offset pair)
 *	@csum_start: Offset from skb->head where checksumming should start
 *	@csum_offset: Offset from csum_start where checksum should be stored
 *	@priority: Packet queueing priority
 *	@ignore_df: allow local fragmentation
 *	@cloned: Head may be cloned (check refcnt to be sure)
 *	@ip_summed: Driver fed us an IP checksum
 *	@nohdr: Payload reference only, must not modify header
 *	@pkt_type: Packet class
 *	@fclone: skbuff clone status
 *	@ipvs_property: skbuff is owned by ipvs
 *	@offload_fwd_mark: Packet was L2-forwarded in hardware
 *	@offload_l3_fwd_mark: Packet was L3-forwarded in hardware
 *	@tc_skip_classify: do not classify packet. set by IFB device
 *	@tc_at_ingress: used within tc_classify to distinguish in/egress
 *	@tc_redirected: packet was redirected by a tc action
 *	@tc_from_ingress: if tc_redirected, tc_at_ingress at time of redirect
 *	@peeked: this packet has been seen already, so stats have been
 *		done for it, don't do them again
 *	@nf_trace: netfilter packet trace flag
 *	@protocol: Packet protocol from driver
 *	@destructor: Destruct function
 *	@tcp_tsorted_anchor: list structure for TCP (tp->tsorted_sent_queue)
 *	@_nfct: Associated connection, if any (with nfctinfo bits)
 *	@nf_bridge: Saved data about a bridged frame - see br_netfilter.c
 *	@skb_iif: ifindex of device we arrived on
 *	@tc_index: Traffic control index
 *	@hash: the packet hash
 *	@queue_mapping: Queue mapping for multiqueue devices
 *	@pfmemalloc: skbuff was allocated from PFMEMALLOC reserves
 *	@active_extensions: active extensions (skb_ext_id types)
 *	@ndisc_nodetype: router type (from link layer)
 *	@ooo_okay: allow the mapping of a socket to a queue to be changed
 *	@l4_hash: indicate hash is a canonical 4-tuple hash over transport
 *		ports.
 *	@sw_hash: indicates hash was computed in software stack
 *	@wifi_acked_valid: wifi_acked was set
 *	@wifi_acked: whether frame was acked on wifi or not
 *	@no_fcs:  Request NIC to treat last 4 bytes as Ethernet FCS
 *	@csum_not_inet: use CRC32c to resolve CHECKSUM_PARTIAL
 *	@dst_pending_confirm: need to confirm neighbour
 *	@decrypted: Decrypted SKB
 *	@napi_id: id of the NAPI struct this skb came from
 *	@secmark: security marking
 *	@mark: Generic packet mark
 *	@vlan_proto: vlan encapsulation protocol
 *	@vlan_tci: vlan tag control information
 *	@inner_protocol: Protocol (encapsulation)
 *	@inner_transport_header: Inner transport layer header (encapsulation)
 *	@inner_network_header: Network layer header (encapsulation)
 *	@inner_mac_header: Link layer header (encapsulation)
 *	@transport_header: Transport layer header
 *	@network_header: Network layer header
 *	@mac_header: Link layer header
 *	@tail: Tail pointer
 *	@end: End pointer
 *	@head: Head of buffer
 *	@data: Data head pointer
 *	@truesize: Buffer size
 *	@users: User count - see {datagram,tcp}.c
 *	@extensions: allocated extensions, valid if active_extensions is nonzero
 */

struct sk_buff {
	union
		struct
			struct sk_buff		*next;
			struct sk_buff		*prev;
			union
				struct net_device	*dev;
				unsigned long		dev_scratch;
		struct rb_node		rbnode; /* used in netem, ip4 defrag, and tcp stack */
		struct list_head	list;

	union
		struct sock		*sk;
		int			ip_defrag_offset;

	union
		ktime_t		tstamp;
		u64		skb_mstamp_ns; /* earliest departure time */
  char			cb[48] __aligned(8);  
	union
		struct
			unsigned long	_skb_refdst;
			void		(*destructor)(struct sk_buff *skb);
		struct list_head	tcp_tsorted_anchor;

	unsigned long		 _nfct;
	unsigned int		len, data_len;
	__u16			mac_len, hdr_len;

	/* Following fields are _not_ copied in __copy_skb_header()
	 * Note that queue_mapping is here mostly to fill a hole.
	 */
	__u16			queue_mapping;

	__u8			__cloned_offset[0];
	__u8			cloned:1, nohdr:1, fclone:2, peeked:1, head_frag:1, pfmemalloc:1;
	/* private: */
	__u32			headers_start[0];
	/* public: */
	__u8			__pkt_type_offset[0];
	__u8			pkt_type:3, ignore_df:1, nf_trace:1, ip_summed:2, ooo_okay:1;
	__u8			l4_hash:1,sw_hash:1,wifi_acked_valid:1,wifi_acked:1,no_fcs:1,encapsulation:1,encap_hdr_csum:1,csum_valid:1

	__u8			__pkt_vlan_present_offset[0];
	__u8			vlan_present,csum_complete_sw,csum_level:2,csum_not_inet,dst_pending_confirm,ndisc_nodetype:2,ipvs_property,
  __u8      inner_protocol_type,remcsum_offload,offload_fwd_mark,offload_l3_fwd_mark,tc_skip_classify,tc_at_ingress,tc_redirected:tc_from_ingress,decrypted
	__u16			tc_index;	/* traffic control index */
	union
		__wsum		csum;
		struct
			__u16	csum_start;
			__u16	csum_offset;

	__u32			priority;
	int			skb_iif;
	__u32			hash;
	__be16			vlan_proto;
	__u16			vlan_tci;
	__u32		secmark;
	union
		__u32		mark;
		__u32		reserved_tailroom;

	union
		__be16		inner_protocol;
		__u8		inner_ipproto;

	__u16			inner_transport_header;
	__u16			inner_network_header;
	__u16			inner_mac_header;
	__be16		protocol;
	__u16			transport_header;
	__u16			network_header;
	__u16			mac_header;

	/* private: */
	__u32			headers_end[0];
	/* public: */
	/* These elements must be at the end, see alloc_skb() for details.  */
	sk_buff_data_t		tail;
	sk_buff_data_t		end;
	unsigned char		*head,*data;
	unsigned int		truesize;
	refcount_t		users;
};



struct skb_shared_info {
	__u8		__unused;
	__u8		meta_len;
	__u8		nr_frags;
	__u8		tx_flags;
	unsigned short	gso_size;
	unsigned short	gso_segs;
	struct sk_buff	*frag_list;
	struct skb_shared_hwtstamps hwtstamps;
	unsigned int	gso_type;
	u32		tskey;
	atomic_t	dataref;
	void *		destructor_arg;
	skb_frag_t	frags[MAX_SKB_FRAGS];
};

```
From book _Linux Device Drivers_
* `struct net_device *rx_dev, *dev`
The devices receiving and sending this buffer, respectively.

From [David Miller's blog](http://vger.kernel.org/~davem/skb.html)
`struct net_device	*dev, *input_dev, *real_dev;`
These three members help keep track of the devices assosciated with a packet. The reason we have three different device pointers is that the main 'skb->dev' member can change as we encapsulate and decapsulate via a virtual device.

So if we are receiving a packet from a device which is part of a bonding device instance, initially 'skb->dev' will be set to point the real underlying bonding slave. When the packet enters the networking (via 'netif_receive_skb()') we save 'skb->dev' away in 'skb->real_dev' and update 'skb->dev' to point to the bonding device.

Likewise, the physical device receiving a packet always records itself in 'skb->input_dev'. In this way, no matter how many layers of virtual devices end up being decapsulated, 'skb->input_dev' can always be used to find the top-level device that actually received this packet from the network.

From Linux 5.x source code, there is only one `struct net_device *dev` and its comment `@dev: Device we arrived on/are leaving by`.

* `union { /* ... */ } h; , union { /* ... */ } nh; , union { /*... */} mac;`
Pointers to the various levels of headers contained within the packet. Each field of the unions is a pointer to a different type of data structure. `h` hosts pointers to transport layer headers (for example, `struct tcphdr *th`); `nh` includes network layer headers (such as `struct iphdr *iph`); and `mac` collects pointers to link layer headers (such as `struct ethdr *ethernet`). Note that network drivers are responsible for setting the mac pointer for incoming packets. This task is normally handled by `ether_type_trans`, but non-Ethernet drivers will have to set `skb->mac.raw` directly.

In Linux 5.x, `union h, nh, mac` became `transport_header network_header mac_header`

* `unsigned char *head; , unsigned char *data; , unsigned char *tail; , unsigned char *end;`
Pointers used to address the data in the packet. _head is the beginning of the allocated space_, _data the beginning of the valid octets_ , _tail is the end of the valid octets_, and _end points to the maximum address tail can reach_. Another way to look at it is that the available buffer space is `skb->end - skb->head`, and the currently used data space is `skb->tail - skb->data`.

[David Miller's skbuff size blog](http://vger.kernel.org/~davem/skb_data.html)

* `unsigned long len;`
The length of the data itself (`skb->tail - skb->data`).

* `data_len`
`skb->data_len` tells how many bytes of paged data (from file system) there are in the _skb_.
`skb_headlen() ==>  skb->len - skb->data_len`

when there is paged data, the packet begins at `skb->data` for `skb_headlen()` bytes, then continues on into the paged data area for `skb->data_len` bytes. That is why it is illogical to try and do a `skb_put()` when there is paged data. You have to add data onto the end of the paged data area instead

* `_skb_refdst`
This member is the generic route for the packet. It tells us how to get the packet to it's destination

* `char cb[48]`
This is the SKB control block. It is an opaque storage area usable by protocols, and even some drivers, to store private per-packet information. TCP uses this, for example, to store sequence numbers and retransmission state for the frame.



* `struct sk_buff *alloc_skb(unsigned int len, int priority);` & `void kfree_skb(struct sk_buff *skb);`
Allocate a buffer and free a buffer.

* `unsigned char *skb_put(struct sk_buff *skb, int len); unsigned char *__skb_put(struct sk_buff *skb, int len);`
These inline functions update the tail and len fields of the sk_buff structure; they are used to add data to the end of the buffer. Each function’s return value is the previous value of skb->tail


* `unsigned char *skb_push(struct sk_buff *skb, int len); unsigned char *__skb_push(struct sk_buff *skb, int len);`
These functions decrement skb->data and increment skb->len. They are similar to skb_put, except that data is added to the beginning of the packet instead of the end. The return value points to the data space just created.

* `unsigned char *skb_pull(struct sk_buff *skb, int len);`
Removes data from the head of the packet.

* `int skb_tailroom(struct sk_buff *skb);` & `int skb_headroom(struct sk_buff *skb);`
This function returns the amount of space available for putting data in the buffer & the amount of space available in front of data

* `void skb_reserve(struct sk_buff *skb, int len);`
This function increments both data and tail.
