## VFIO

### [KVM MMU AT Linux Doc](https://elixir.bootlin.com/linux/v2.6.35/source/Documentation/kvm/mmu.txt)


### [VFIO linux doc](https://www.kernel.org/doc/Documentation/vfio.txt)
The VFIO driver is an IOMMU/device agnostic framework for exposing direct device access to userspace, in a secure, IOMMU protected environment.

Why do we want that?  Virtual machines often make use of direct device access ("device assignment") when configured for the highest possible I/O performance.  From a device and host perspective, this simply turns the VM into a userspace driver

The VFIO driver framework intends to unify these, replacing both the KVM PCI specific device assignment code as well as provide a more secure, more featureful userspace driver environment than UIO.
