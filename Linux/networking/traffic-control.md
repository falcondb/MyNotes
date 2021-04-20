## Traffic control
[Traffic Control HOWTO](https://tldp.org/HOWTO/Traffic-Control-HOWTO/overview.html)

### What is Traffic control
> Traffic control is the set of tools which allows the user to have granular control over ingress & egress queues and the queuing mechanisms of a networked device.

> The term Quality of Service (QoS) is often used as a synonym for traffic control.

> you can think of traffic control as a way to provide some of the statefulness of a circuit-based network to a packet-switched network.

#### Tokens and buckets
> Instead of calculating the current usage and time, one method is to generate tokens at a desired rate, and only dequeue packets or bytes if a token is available.

> Used in TBF and HTB

#### Traditional Elements of Traffic Control
* [TBF, Token Bucket Filter](https://tldp.org/HOWTO/Traffic-Control-HOWTO/classless-qdiscs.html#qs-tbf)
* [HTB, Hierarchical Token Bucket](https://tldp.org/HOWTO/Traffic-Control-HOWTO/classful-qdiscs.html#qc-htb)
* [Shaping, Scheduling, Classifying, etc](https://tldp.org/HOWTO/Traffic-Control-HOWTO/elements.html#e-shaping)
  * _Shapers_ delay packets to meet a desired rate. Shapers attempt to limit or ration traffic to meet but not exceed a configured rate.
  * _Scheduling_ is the mechanism by which packets are arranged (or rearranged) between input and output of a particular queue (e.g., FIFO).
  * _Classifying_ is the mechanism by which packets are separated for different treatment, possibly different output queues.
  * _Policers_ measures and limits traffic in a particular queue. Policing is most frequently used on the network border to ensure that a peer is not consuming more than its allocated bandwidth
  * _Dropping_
  * _Marking_ installs a DSCP on the packet itself, which is then used and respected by other routers inside an administrative domain

#### qdisc
A qdisc is a scheduler, called a queuing discipline.
> The classful qdiscs can contain classes, and provide a handle to which to attach filters.

> root qdisc and ingress qdisc are not really queuing disciplines, but rather locations onto which traffic control structures can be attached for egress and ingress.

> Traffic transmitted on an interface traverses the egress or root qdisc.

> For practical purposes, the ingress qdisc is merely a convenient object onto which to attach a policer to limit the amount of traffic accepted on a network interface.

#### classes
> Any class can also have an arbitrary number of filters attached to it, which allows the selection of a child class or the use of a filter to reclassify or drop traffic entering a particular class.

> A leaf class is a terminal class in a qdisc. It contains a qdisc (default FIFO) and will never contain a child class.

#### filter
> The filter provides a convenient mechanism for gluing together several of the key elements of traffic control. The simplest and most obvious role of the filter is to classify packets. Linux filters allow the user to classify packets into an output queue with either several different filters or a single filter.

> A filter _must_ contain a classifier phrase. A filter may contain a policer phrase.

> Filters can be attached either to classful qdiscs or to classes, however the enqueued packet always enters the root qdisc first. After the filter attached to the root qdisc has been traversed, the packet may be directed to any subclasses


#### classifier
Filter objects can be manipulated using tc. The classifiers are tools which can be used as part of a filter to identify characteristics of a packet or a packet's metadata.

#### policer
This elemental mechanism is only used in Linux traffic control as part of a filter. A policer calls one action above and another action below the specified rate.

#### handler
Every class and classful qdisc requires a unique identifier, _handler_.
The handle is used as the target in classid and flowid phrases of `tc` filter statements. These handles are external identifiers for the objects, usable by userland applications.

### Classless Queuing Disciplines

* FIFO
  * pfifo - Packet limited First In, First Out queue
  * bfifo - Byte limited First In, First Out queue
* pfifo_fast
  * the default qdisc for all interfaces under Linux
  * three different prioritized bands (individual FIFOs, from band # 0 to # 2) for separating traffic
  * Three methods are available to PRIO to determine in which band a packet will be enqueued.
    * _From userspace_, `setsockopt` with `SO_PRIORITY`
    * _With a `tc` filter_, a `tc` filter attached to the root qdisc can point traffic directly to a class
    * _With the priomap_, Based on the packet priority, which in turn is derived from the _Type of Service_ assigned to the packet.

* SFQ, Stochastic Fair Queuing
  * a hash function to separate the traffic into separate FIFOs which are dequeued in a round-robin fashion.
  * Because there is the possibility for unfairness to manifest in the choice of hash function, this function is altered periodically.

* ESFQ, Extended Stochastic Fair Queuing
  * Conceptually, this qdisc is no different than SFQ although it allows the user to control more parameters than its simpler cousin
* GRED, Generic Random Early Drop

* TBF, Token Bucket Filter
  * Packets are only transmitted if there are sufficient tokens available. Otherwise, packets are deferred

* HTB, Hierarchical Token Bucket
  * allows the user to define the characteristics of the tokens and bucket used and allows the user to nest these buckets in an arbitrary fashion
  * Children classes borrow tokens from their parents once they have exceeded rate. A child class will continue to attempt to borrow until it reaches ceil, at which point it will begin to queue packets for transmission until more tokens/ctokens are available.
    * leaf	< rate	HTB_CAN_SEND	Leaf class will dequeue queued bytes up to available tokens (no more than burst packets)
    * leaf	> rate, < ceil	HTB_MAY_BORROW	Leaf class will attempt to borrow tokens/ctokens from parent class. If tokens are available, they will be lent in quantum increments and the leaf class will dequeue up to cburst bytes
    * leaf	> ceil	HTB_CANT_SEND	No packets will be dequeued. This will cause packet delay and will increase latency to meet the desired rate.
    * inner, root	< rate	HTB_CAN_SEND	Inner class will lend tokens to children.
    * inner, root	> rate, < ceil	HTB_MAY_BORROW	Inner class will attempt to borrow tokens/ctokens from parent class, lending them to competing children in quantum increments per request.
    * inner, root	> ceil	HTB_CANT_SEND	Inner class will not attempt to borrow from its parent and will not lend tokens/ctokens to children classes.

  * HFSC, Hierarchical Fair Service Curve, HFSC, Hierarchical Fair Service Curve, CBQ, Class Based Queuing

### Rules, Guidelines and Approaches
* Any router performing a shaping function should be the bottleneck on the link, and should be shaping slightly below the maximum available link bandwidth.
* A device can only shape traffic it transmits.
* Every interface must have a qdisc (default is pfifo_fast).
* One of the classful qdiscs added to an interface with no children classes typically only consumes CPU for no benefit.
* Any newly created class contains a FIFO. The FIFO qdisc will be removed implicitly if a child class is attached to this class.
* Classes directly attached to the root qdisc can be used to simulate virtual circuits.
* A filter can be attached to classes or one of the classful qdiscs.
* HTB is an ideal qdisc to use on a link with a known bandwidth
* In theory, the PRIO scheduler is an ideal match for links with variable bandwidth

### direct action (da) and clsact
[Understanding tc “direct action” mode for BPF](https://qmonnet.github.io/whirl-offload/2020/04/11/tc-bpf-direct-action/)
As classifiers, eBPF brings more flexibility for parsing programs, and even allows stateful processing or interaction with user-space. they return a value that can be 0 for a mismatch, -1 for a match, or any other class identifier.

eBPF classifiers alone are enough to filter and process the packets, and do not need additional qdiscs or classes to be attached to them. To avoid to add such simple TC actions and to simplify those use cases where the classifier does all the work, a new flag was added to TC for eBPF classifiers: direct-action, also available as _da_ for short.  This flag tells the system that the return value from the filter should be considered as the one of an action instead _TC_ACT_SHOT_, _TC_ACT_OK_

eBPF actions could still be used after other filters.

the special field `tc_classid` of the `struct __skb_buff` can be used, instead of the program return value, to tell the system where to send the packet when the program allows it to pass

_clsact_ is similar to the ingress qdisc, to which we can attach eBPF programs with the direct-action mode, and which does not perform any queuing. But _clsact_ acts as a superset of ingress

[Linux Advanced Routing & Traffic Control HOWTO](https://www.lartc.org/lartc.html#LARTC.QDISC.TERMINOLOGY)

[Monitoring and Tuning the Linux Networking Stack: Sending Data](https://blog.packagecloud.io/eng/2017/02/06/monitoring-tuning-linux-networking-stack-sending-data/)
For network hardware that supports only a single transmit queue, the default qdisc _pfifo_fast_ is used. Network hardware that supports multiple transmit queues uses the default qdisc of _mq_. You can check your system by running `tc qdisc`.
