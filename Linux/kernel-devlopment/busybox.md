## Busybox

### Download Busybox
```
wget https://busybox.net/downloads/binaries/1.21.1/busybox-x86_64 -O busybox

# or
sudo apt install busybox -y

```


```
!/bin/busybox sh

busybox/mkdir -p ${WORKPATH}/initramfs/{usr/sbin,usr/bin,sbin,bin}
busybox --install -s
```
