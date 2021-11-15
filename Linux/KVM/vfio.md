
### VIRTIO AT IBM
#### [Virtio: An I/O virtualization framework for Linux](https://developer.ibm.com/articles/l-virtio/)
  - virtio is an abstraction for a set of common emulated devices in a paravirtualized hypervisor. This design allows the hypervisor to export a common set of emulated devices and make them available through a common application programming interface
  - With paravirtualized hypervisors, the guests implement a common set of interfaces, with the particular device emulation behind a set of back-end drivers. The back-end drivers need not be common as long as they implement the required behaviors of the front end

  - Concept hierarchy
    - _virtio_driver_ represents the front-end driver in the guest
    - _virtio_device_ To identify the _virtqueues_ that associate with this _virtio_device_, use the _virtio_config_ops_ object with the _find_vq_ function
    - _virtio_config_ops_ for configuring the virtio device
    - _virtqueue_
      - _virtqueue_ops_
  - Core API
    - The virtqueue supports its own API consisting of five functions
      - `add_buf` in the form of the scatter-gather list (buffers for in for response & out with data). notify the hypervisor using the _kick_ function
      - `get_buf` poll or wait for call back.
      - `enable_cb / disable_cb`

### VIRTIO AT Redhat
#### [Virtio Devices High-Level Design](https://projectacrn.github.io/latest/developer-guides/hld/hld-virtio-devices.html)
The virtio APIs can be divided into 3 groups:
  - DM APIs are exported by the DM, and are mainly used during the device initialization phase and runtime. The DM APIs also include PCIe emulation APIs because each virtio device is a PCIe device in the Service VM and User VM.

  - VBS APIs are mainly exported by the VBS and related modules. Generally they are callbacks to be registered into the DM.

  - VQ APIs are used by a virtio backend device to access and parse information from the shared memory between the frontend and backend device drivers.

Key Concepts
  - Frontend virtio driver (FE)
    - The FE driver merely needs to offer services configure the interface, pass messages, produce requests, and kick backend virtio driver.
  - Backend virtio driver (BE)
    - the BE driver, running either in user-land or kernel-land of the host OS, consumes requests from the FE driver and sends them to the host native device driver.
    - once the requests are done by the host native device driver, the BE driver notifies the FE driver
  - Straightforward: virtio devices as standard devices on existing buses
    - Instead of creating new device buses from scratch, virtio devices are built on existing buses. This gives a straightforward way for both FE and BE drivers to interact with each other.
    - Currently virtio supports PCI/PCIe bus and MMIO bus
  - Efficient: batching operation is encouraged
  - Standard: virtqueue
    - All virtio devices share a standard ring buffer and descriptor mechanism, called a virtqueue
      - add_buf is for adding a request/response buffer in a virtqueue
      - get_buf is for getting a response/request in a virtqueue
      - kick is for notifying the other side for a virtqueue to consume buffers
    - The virtqueues are created in guest physical memory by the FE drivers. BE drivers only need to parse the virtqueue structures to obtain the requests and process them.
    - Virtio Device Discovery
      - Virtio devices are commonly implemented as PCI/PCIe devices. A virtio device using virtio over PCI/PCIe bus must expose an interface to the Guest OS that meets the PCI/PCIe specifications.  

Virtio Frameworks
  - Architecture
    - Basically the FE and BE driver communicate with each other through shared memory, via the virtqueues.
    - Each side has three layers: transports, core models, and device types. All virtio devices share the same virtio infrastructure, including virtqueues, feature mechanisms, configuration space, and buses.
  - Virtio Framework Considerations
    - In ACRN, the virtio framework implementations can be classified into two types, virtio backend service in user-land (VBS-U) and virtio backend service in kernel-land (VBS-K)
  - User-Land Virtio Framework
    - The FE driver talks to the BE driver as if it were talking with a PCIe device. This means for “control plane”, the FE driver could poke device registers through PIO or MMIO, and the device will interrupt the FE driver when something happens. For “data plane”, the communication between the FE and BE driver is through shared memory, in the form of virtqueues.
    - See the frame architecture diagram
  - Kernel-Land Virtio Framework
    - VBS-K, designed from scratch for ACRN, the other called Vhost, compatible with Linux Vhost
    - Generally VBS-K provides acceleration towards performance critical devices emulated by VBS-U modules by handling the “data plane” of the devices directly in the kernel.
    - VBS-K still relies on VBS-U for feature negotiations between FE and BE drivers. This means the “control plane” of the virtio device still remains in VBS-U
  - Vhost Framework
    - Vhost is a specific kind of virtio where the data plane is put into host kernel space to reduce the context switch
    - The vhost general data plane workflow can be described as:
      - vhost proxy creates two eventfds per virtqueue, one is for kick, (an ioeventfd), the other is for call, (an irqfd).
      - vhost proxy registers the two eventfds to HSM through HSM character device:
        - Ioevenftd is bound with a PIO/MMIO range. If it is a PIO, it is registered with (fd, port, len, value). If it is an MMIO, it is registered with (fd, addr, len).
        - Irqfd is registered with MSI vector.
      - vhost proxy sets the two fds to vhost kernel through ioctl of vhost device.
      - vhost starts polling the kick fd and wakes up when guest kicks a virtqueue, which results a event_signal on kick fd by HSM ioeventfd.
      - vhost device in kernel signals on the irqfd to notify the guest.
    - Ioeventfd Implementation & Irqfd Implementation
      - implemented in HSM  
      - See the diagram in the post


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


### [Virtqueues and virtio ring: How the data travels](https://www.redhat.com/en/blog/virtqueues-and-virtio-ring-how-data-travels)

  - Split virtqueue: the beauty of simplicity
    - The split virtqueue format separates the virtqueue into three areas
      - _Descriptor Area_: used for describing buffers
      - _Driver Area_: data supplied by driver to the device. Also called _avail virtqueue_
      - _Device Area_:data supplied by device to driver. Also called _used virtqueue_
    - They need to be allocated in the driver’s memory for it to be able to access them in a straightforward way. Buffer addresses are stored from the driver's point of view, and the device needs to perform an address translation.
      - For an emulated device in the hypervisor (like qemu), the guest's address is in its own process memory (by shadow paging?)
      - For other emulated devices like vhost-net or vhost-user, a memory mapping needs to be done, like POSIX shared memory. A file descriptor to that memory is shared through vhost protocol.
      - For a real device a hardware-level translation needs to be done, usually via _IOMMU_
    - Descriptor ring: contains an array of a number of guest addressed buffers and its length, and  a set of flags (chained descripter, read-only/write-only)  
    - Avail ring: The `idx` field indicates where the driver would put the next descriptor entry; `flags` the least significant bit of flags indicates if the driver wants to be notified
    - Used ring: return the used buffers
    - Indirect descriptors: dispatch a larger number of descriptors in a batch, increasing the ring capacity.


### [Virtualization technology implementation-KVM I/O virtualization](https://programmersought.com/article/38004209835/)
  - the storage device of a virtual machine is simulated by QEMU and can be divided into several parts:
    - The top virtual machine disk: e.g. virtio-blk disk, virtio-scsi disk, ide disk. When the virtual machine reads and writes data to these disks, QEMU obtains the IO/SCSI/ATA request of the virtual machine through the top disk and converts it into specific disk data reading and writing;
    - Middle layer virtual machine image layer: e.g. raw, qcow2, sheepdog and other format type images. In fact, the data is placed according to specific protocol rules to achieve some advanced features of storage (e.g. linking, cloning, snapshots, etc.). The sector order of the data has already been inconsistent with the original data through this layer;
    - The bottom layer adapts to different and specific storage files or devices on the Host: the middle layer (mirror layer) has rearranged the data, and the data is written/read on the bottom layer to the real files or devices on the Host.

### [eventfd in qemu-ioeventfd](https://www.programmersought.com/article/83465096784/)

### [IO virtualization - virtio-blk front-end driver analysis](https://programmersought.com/article/58091005566/)

### [IO virtualization - virtio introduction and code analysis](https://programmersought.com/article/11831007453/)
