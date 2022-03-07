## initramfs

* Kernel configuration: `CONFIG_BLK_DEV_INITRD`
* Embedded initramfs in kernel image: ` CONFIG_INITRAMFS_SOURCE`
* GRUB configuration for initrd: `initrd $INITRAM_ACHIEVE` in `/boot/grub/grub.cfg`

* `usr/gen_initramfs.sh` creates a cpio list by analyzing all directories and files within the initram source directory
* `usr/gen_init_cpio` generates the cpio archive file from the list
* `gzip` the cpio archive file

* `mkinitramfs` generates an initramfs image, see `man mkinitramfs`
* `dracut` generates an initramfs/initrd image, `--include` to add addition files to the image
* `lsinitrd` lists the context of an image

#### [Root FS introduction by Rob Landley](https://landley.net/writing/rootfs-intro.html)
A history of the root disk, ram-based initial root filesystem, ram based block device,

#### [Tech Tip: How to use initramfs by Rob Landley](https://landley.net/writing/rootfs-howto.html)

Shortly after mounting rootfs during bootup, the kernel extracts a gzipped cpio archive into it. Then the kernel tries to run "/init" out of rootfs, if that works the kernel is done booting.

##### An example of creating your own initramfs archive
```
# Write a hello world code, hang inside as an init toy example

gcc -static *.c -o init

mkdir sub
cp init sub/init
cd sub

find . | cpio -o -H newc | gzip --best > ../initramfs_data.cpio.gz

# Add the archive to kernel image by CONFIG_INITRAMFS_SOURCE
# If CONFIG_INITRAMFS_SOURCE points to a dir, it does: usr/gen_initramfs_list.sh $CONFIG_INITRAMFS_SOURCE > usr/initramfs_list; usr/gen_init_cpio usr/initramfs_list > usr/initramfs_data.cpio; gzip usr/initramfs_data.cpio
# Or copy it to usr/initramfs_data.cpio.gz in the kernel build dir

```

#### [Programming for Initramfs](https://landley.net/writing/rootfs-programming.html)
There is a script of creating the given executables and their shared-libraries to a rootfs

Busybox staff


#### [Custom Initramfs](https://wiki.gentoo.org/wiki/Custom_Initramfs#Busybox)

```
mkdir -p ${YOURWORKINGPATH}/initramfs/{bin,dev,etc,lib,lib64,mnt/root,proc,root,sbin,sys}

## Support /dev/sda1 is needed in init in initramfs
cp --archive /dev/{null,console,tty,sda1} ${YOURWORKINGPATH}/initramfs/dev/

```

An script of mounting root fs using busybox
```
#!/bin/busybox sh

mount -t proc none /proc
mount -t sysfs none /sys

mount -o ro ${YOURROOTDEV} ${ROOTPATH}

umount /proc
umount /sys

exec switch_root ${ROOTPATH} /sbin/init

```

Mode complex initram [examples](https://wiki.gentoo.org/wiki/Custom_Initramfs/Examples)

#### [genkernel@Gentoo Linux](https://wiki.gentoo.org/wiki/Genkernel)

#### [Dracut@Gentoo Linux](https://wiki.gentoo.org/wiki/Dracut)

#### [Dracut@Fedora Linux](https://fedoraproject.org/wiki/Dracut)
