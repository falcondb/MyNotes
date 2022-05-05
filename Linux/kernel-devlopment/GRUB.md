## GRUB (GRand Unified Bootloader)

### Key files
- `/usr/sbin/grub-mkconfig`: generates the configuration file, e.g., `/boot/grub/grub.cfg`
- `/boot/grub/grub.cfg`: the configuration read by Grub interpreter
- `/etc/grub.d/`: shell scripts to generate the configuration file
  - `00_header`: reads variables from `/etc/default/grub`
  - `10_linux`: reads menu entries for Linux
- `/boot/grub/custom.cfg`: additional commands during boot time
- `/etc/default/grub`: variables used to generate the configuration file
- `update-grub`: `set -e; exec grub-mkconfig -o /boot/grub/grub.cfg "$@"`

### Configuration parameters

Set in `/etc/default/grub`
- `GRUB_TIMEOUT`
- `GRUB_CMDLINE_LINUX`: Parameters to the kernel command line
- `GRUB_DEVICE`: the initial root device


### Commands
`grub2-install /dev/sda` install grub in the MBR of the sda drive


### libvirt
In order to virsh to the console of the VM, the `/etc/default/grub` needs these configurations:
  - `GRUB_CMDLINE_LINUX_DEFAULT="console=ttyS0,115200n8"`
  - `GRUB_TERMINAL=serial`

[Reference](https://bobcares.com/blog/virsh-console-not-working/)
