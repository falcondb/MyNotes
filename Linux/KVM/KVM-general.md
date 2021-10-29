### VIRTIO
#### [Introduction to virtio-networking and vhost-net](https://www.redhat.com/en/blog/introduction-virtio-networking-and-vhost-net)
We can split virito into two parts:
  - _virtio spec_: The virtio spec, which is maintained by OASIS, defines how to create a control plane and the data plane between the guest and host. For example the data plane is composed of buffers and rings layouts which are detailed in the spec.

  - _vhost protocol_: A protocol that allows the virtio dataplane implementation to be offloaded to another element in order to enhance performance.

If we simply implemented the virtio spec data plane in qemu we’d have a context switch for every packet going from the kernel to the guest, and vice versa. This is where the vhost protocol comes into play, enabling us to implement a data plane going directly from the kernel (host) to the guest bypassing the qemu process. The vhost protocol itself only describes how to establish the data plane, however. Whoever implements it is also expected to implement the ring layout for describing the data buffers (both host and guest) and the actual send/receive packets.  

The vhost-net/virtio-net architecture
  - vhost-net is the backend running in the host kernel space; virtio-net is the frontend running in the guest kernel space
  - Vhost-net uses the vhost protocol to establish the framework which is then used for the data plane to directly forward packets between the host and guest kernel using a shared memory area. In practice the data plane communication, receive (RX) and transmit (TX), is accomplished through dedicated queues.

Virtio-networking and OVS
   - In order to forward these packets to other guest running on the same host or outside the hosts we use OVS.
   - _OVS_ is a software switch which enables the packet forwarding inside the kernel. It’s composed of a userspace part and a kernel part:
    - User space: including a database (ovsdb-server) and an OVS daemon for managing and controlling the switch (ovs-vswitchd).
    - Kernel space: including the ovs kernel module responsible for the datapath or forwarding plane.
   - The OVS controller communicates both with the database server and the kernel forwarding plane. To push packets in and out of the OVS we use Linux ports. In our case we have one port that connects the OVS kernel forwarding plane to a physical NIC while the other port connects to the vhost-net backend. See the architecture diagram in this post

#### [Deep dive into Virtio-networking and vhost-net](https://www.redhat.com/en/blog/deep-dive-virtio-networking-and-vhost-net)
Virtio specification: devices and drivers
  - It uses the fact that the guest can share memory with the host for I/O to implement that
  - The virtio specification is based on two elements: devices and drivers. In a typical implementation, the hypervisor exposes the virtio devices to the guest through a number of transport methods. The most common transport method is PCI or PCIe bus. However, the device can be available at some predefined guest’s memory address.
  - Real PCI hardware exposes its configuration space using a specific physical memory address range. In the VM world, the hypervisor captures accesses to that memory range and performs device emulation, exposing the same memory layout that a real machine would have and offering the same responses.
  - The virtio kernel drivers share a generic transport-specific interface (e.g: virtio-pci), used by the actual transport and device implementation (such as virtio-net, or virtio-scsi).
Virtio specification: virtqueues
  - Virtqueues are the mechanism for bulk data transport on virtio devices. Each device can have zero or more virtqueues.
  - the virtio specification also defines bi-directional notifications: Available Buffer Notification; Used Buffer Notification.
  - After filling the packet to be sent, it triggers an "available buffer notification", returning the control to QEMU so it can send the packet through the TAP device.  QEMU then notifies the guest that the buffer operation (reading or writing) is done, and it does that by placing the data in the virtqueue and sending a used notification event, triggering an interruption in the guest vCPU.
  - The process of receiving a packet is similar to that of sending it.
  - Refer to the flow diagram in the post.

Vhost protocol
  - The vhost API is a message based protocol that allows the hypervisor to offload the data plane to another component (handler) that performs data forwarding more efficiently.
  - the master sends the following configuration information to the handler: The hypervisor’s memory layout; A pair of file descriptors that are used for the handler to send and receive the notifications defined in the virtio spec. These file descriptors are shared between the handler and KVM so they can communicate directly without requiring the hypervisor’s intervention.

Vhost-net
  - The vhost-net is a kernel driver that implements the handler side of the vhost protocol to implement an efficient data plane. QEMU and the vhost-net kernel driver (handler) use ioctls to exchange vhost messages and a couple of eventfd-like file descriptors called irqfd and ioeventfd are used to exchange notifications with the guest.
  - When vhost-net kernel driver is loaded, it exposes a character device on `/dev/vhost-net`.
  - During the initialization the vhost-net kernel driver creates a kernel thread called vhost-$pid, where $pid is the hypervisor process pid. This thread is called the "vhost worker thread". A tap device is still used to communicate the VM with the host but now the worker thread handles the I/O events
  - QEMU allocates one eventfd and registers it to both vhost and KVM in order to achieve the notification bypass. The vhost-$pid kernel thread polls it, and KVM writes to it when the guest writes in a specific address. This mechanism is named _ioeventfd_. It does not need to go through QEMU.
  - On the other hand, QEMU allocates another eventfd and registers it to both KVM and vhost again for direct vCPU interruption injection. This mechanism is called _irqfd_, and it allows any process in the host to inject vCPU interrupts to the guest by writing to it

Communication with the outside world
  - How does it communicate with other VMs on the same host or with machines outside the host? Linux bridges or Open Virtual Switch (OVS).
  - OVS datapath is running in the kernel and is forwarding the packets between the physical NIC and the virtual TAP device


#### [Hands on vhost-net: Do. Or do not. There is no try](https://www.redhat.com/en/blog/hands-vhost-net-do-or-do-not-there-no-try)


#### [How vhost-user came into being: Virtio-networking and DPDK](https://www.redhat.com/en/blog/how-vhost-user-came-being-virtio-networking-and-dpdk)
DPDK overview
  - It implements a "run to completion model for packet processing" meaning that all resources need to be allocated prior to calling the data plane application. Those dedicated resources are executed on dedicated logical processing cores.
  - In the DPDK architecture the devices are accessed by constant polling. This avoids the context switching and interrupt processing overhead at the cost of dedicating 100% of part of the CPU cores to handle packet processing
  - In practice DPDK offers a series of poll mode drivers (PMDs) that enable direct transfer of packets between user space and the physical interfaces which bypass the kernel network stack all together.
  -
