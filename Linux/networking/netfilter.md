## NetFilter
[netfilter.org](http://www.netfilter.org/index.html)


## IPtables
[A Deep Dive into Iptables and Netfilter Architecture @digitalocean](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)
[iptables @netfilter.org](http://www.netfilter.org/projects/iptables/index.html)

## Connection Tracking

[Netfilter’s connection tracking system](http://wiki.netfilter.org/pablo/docs/login.pdf)
* Hooks
  * _PREROUTING_: All the packets, with no exceptions, hit this hook, which is reached before the routing decision and after all the IP header sanity checks are fulfilled. _Port Address Translation (NAPT)_ and _Redirections_, that is, _Destination Network Translation (DNAT)_, are implemented in this hook.
  * _LOCAL INPUT_: All the packets going to the local machine reach this hook. This is the _last_ hook in the incoming path for the local machine traffic.
  * _FORWARD_: Packets not going to the local machine reach this hook.
  * _LOCAL OUTPUT_: This is the _first_ hook in the outgoing packet path. Packets leaving the local machine always hit this hook.
  * _POSTROUTING_: This hook is implemented after the routing decision. _Source Network Address Translation (SNAT)_ is registered to this hook. All the packets that leave the local machine reach this hook.

* Conntrack states
  * _NEW_: The connection is starting. This state is reached if the packet is valid, that is, if it belongs to the valid sequence of initialization, and if the firewall has only seen traffic in one direction.
  * _ESTABLISHED_: The connection has been established. In other words, this state is reached when the firewall has seen two-way communication.
  * _RELATED_: This is an expected connection.
  * _INVALID_: This is a special state used for packets that do not follow the expected behavior of a connection.

* Connection hash tables
There are two hash tuples for every connection: one for the original direction; one for the reply direction.
A tuple represents the relevant information of a connection. Such tuples are embedded in a hash tuple. The two hash tuples are embedded in the structure nf_conn.

* DEFRAGMENTED PACKET HANDLING
By the callback `ipv4_conntrack_defrag`. which gathers the defragmented packets. Once they are successfully received, the fragments continue their travel through the stack

* HELPERS AND EXPECTATIONS
  * _Helpers_ lets the system identify whether a connection is related to an existing one. An _expectation_ is a connection that is expected to happen in a period of time. It is defined as an `nf_conntrack_expect` structure. When the system finds a matching expectation, the new conntrack is related to the master conntrack that created such an expectation.

[Netfilter Hacking How-to](https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO.html)
Only the first package of a new connection will traverse the table: the result of this traversal in then applied to all future packets in the same connection.

After a table is registered with a hook, userspace can read and replace its contents using `getsockopt` and `setsockopt`

The kernel starts traversing the location indicted by the particular hook. That rule is examined, if the `struct ipt_ip` elements match, each `struct ipt_entry_match` is checked in turn. If the match function returns 0, iteration stops on that rule. If the iteration continues to the end, the `struct ipt_entry_target` is examined: if it's a standard target. tje _verdict_ field is read (negtive means a packet verdict, positive means an offset to jump to). IF the answer is positive and the offset is not that of the next rule, the _back_ variable is set, and the previous _back_ value is placed in that rule's `confrom` field.

Kernel module for iptable:
Module code must be re-entrant.

_libiptc_ is the iptables control library, designed for listing and manipulating rules in the iptables kernel module.
```
init_module
cleanup_module
ipt_register_match
ipt_register_target
ipt_unregister_match
ipt_unregister_target
```

`_nfcache` in `skb`: Each `skb` has a `nfcache` field: a bitmask of what fields in the header were examined, and whether the packet was altered or not.

[Connection Tracking (conntrack): Design and Implementation Inside Linux Kernel](http://arthurchiao.art/blog/conntrack-design-and-implementation/)
[Netfilter Connection Tracking and NAT Implementation](https://wiki.aalto.fi/download/attachments/69901948/netfilter-paper.pdf)
