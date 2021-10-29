## [Virtio Devices High-Level Design](https://projectacrn.github.io/latest/developer-guides/hld/hld-virtio-devices.html)
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
