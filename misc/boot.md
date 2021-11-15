### [Stages of Linux Boot Process](https://www.thegeekstuff.com/2011/02/linux-boot-process/)
  - BIOS
    - Once the boot loader program is detected and loaded into the memory, BIOS gives the control to it
  - MBR:
    - in the 1st sector of the bootable disk.
    - three components
      - primary boot loader info in 1st 446 bytes
      - partition table info in next 64 bytes
      - mbr validation check in last 2 bytes.
      - MBR loads and executes the GRUB boot loader
  - GRUB
    - GRUB has the knowledge of the filesystem
    - `/boot/grub/grub.conf` / `/etc/grub.conf`

  - Kernel
    - Mounts the root file system as specified in the `root=` in grub.conf
    - executes the `/sbin/init`, e.g., `systemd`
      - root targe is default.target ==> Graphical.target (`/usr/lib/systemd/system/graphical.target`)
      - multi-user.target (`/etc/systemd/system/multi-user.target.wants`) is for user daemon
      - multi-user.target ==> basic.target (`etc/systemd/system/basic.target.wants`)
      - basic.target      ==> sysinit.target (`/etc/systemd/system/sysinit.target.wants/`)
      - sysinit.target    ==> local-fs.target (`/etc/systemd/system/local-fs.target.wants/`
    - `initrd` is used by kernel as temporary root file system until kernel is booted

  - Init
    - set run level
  - Runlevel programs
    - Depending on your default init level setting, the system will execute the programs from _one_ of the following directories. `Run level 0 – /etc/rc.d/rc0.d/ `
    - Programs starts with S are used during startup. _S_ for startup.
    - Programs starts with K are used during shutdown. _K_ for kill.

### [System bootup process @ systemd.index](https://www.freedesktop.org/software/systemd/man/bootup.html)
