## QEMU Tools

### Qemu commands

* `qemu-<arch>`: userland emulation mode
* `qemu-system-<arch>`: full system emulation mode, including kernel

### Qemu + GDB
Run Qemu traced by GDB, e.g., `qemu-system-x86_64 -M pc -s -S -monitor stdio -display none`


### qemu-nbd

`apt install qemu-utils kpartx -y; modprobe nbd`


```
sudo qemu-nbd -c /dev/nbd${YOURID} $IMAGE

# add partition to dev/nbd
sudo kpartx -a /dev/nbd${YOURID}

# mount your nbd device

# clean up
sudo qemu-nbd -d /dev/nbd0
sudo rmmod nbd
```
