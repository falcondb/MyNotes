[Prepare the environment for developing Linux kernel with qemu](https://medium.com/@daeseok.youn/prepare-the-environment-for-developing-linux-kernel-with-qemu-c55e37ba8ade)

[Linux kernel + QEMU + gdb](http://nickdesaulniers.github.io/blog/2018/10/24/booting-a-custom-linux-kernel-in-qemu-and-debugging-it-with-gdb/)
`-nographic` to disable GUI,  `-append "console=ttyS0"` setup a tty
`mkinitramfs -o ramdisk.img` to create a _initramfs_ image, `qemu-xxxx -initrd ramdisk.img` to use the image for init process

[Kernel Debugging](https://wiki.osdev.org/Kernel_Debugging)

[debugging_kernel.txt](https://gist.github.com/hngouveia01/843a2202628c7d567dad0f657f8373aa)

[QEMU Guide](https://www.qemu.org/docs/master/system/index.html)

* QEMU Monitor section
    `gdbserver [port]` Start gdbserver session (default port=1234)

* Direct Linux Boot section:
  `qemu-system-x86_64 -kernel bzImage -hda rootdisk.img `
  Use `-kernel` to provide the Linux kernel image and `-append` to give the kernel command line arguments. The `-initrd` option can be used to provide an _INITRD_ image. If you do not need graphical output, you can disable it and redirect the virtual serial port and the QEMU monitor to the console with the `-nographic` option.

* GDB usage section
  In order to use gdb, launch QEMU with the -s and -S options. The -s option will make QEMU listen for an incoming connection from gdb on TCP port 1234, and -S will make QEMU not start the guest until you tell it to from gdb.

```
gdb vmlinux

gdb> target remote localhost:1234
```

[QEMU GDB](https://qemu-project.gitlab.io/qemu/system/gdb.html)
