​# Linux kernel debugging through GDB
The followings are the key steps of build, configuration, deployment and debugging Linux kernel
## Build the linux kernel locally
Compile the kernel source code and get the two kernel image files ready locally
    - A compressed kernel image file, at this moment, it is a bzImage file (Linux kernel x86 boot executable bzImage), with name like `vmlinuz-$BUILDVERSION`. It should be at `/boot/` after running `make install`. This file will be used as the kernel image when we run QEMU VM.
    - A ELF binary file with debug_info (when build kernel, set `CONFIG_DEBUG_INFO=y`). It typically at the root of Linux kernel git repo after compliation. Make sure we need the ELF raw file, not the compressed ELF (in `arch/x86/boot/compressed`).

## Run a QEMU VM with the target kernel image
To run a VM with a custom kernel image, I recommend the following configurations
- A root fs from a Linux distribuation like Ubuntu, Centos, etc. The root fs can be prepared by running the Linux distributation first and then save the root fs of the VM on the host (e.g., as a qcow2 file)
- Now run a QEMU VM with the kernel image we built
  - `-hda` to the root fs we prepared
  - `-kernel` to the kernel bzImage file
  - `-append "root=/dev/sdaX console=ttyS0,115200 nokaslr", the `X` in `sdaX` is the root fs located as a drive in the VM (figure out this device ID by running the VM without the custom kernel image)
  - `-s` and `-S`
  - `-enable-kvm`
  - IO setting, e.g, network
Configuration the VM is ready, run the QEMU VM.

## Run GDB against the QEMU VM
- `target remote :1234` `set architecture i386:x86-64:intel`
Should have a live debugger attached to the kernel
