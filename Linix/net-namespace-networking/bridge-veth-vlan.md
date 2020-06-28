## Origin
[From Linux-Blog – Dr. Mönchmeyer's Post](https://linux-blog.anracom.com/2017/11/28/fun-with-veth-devices-linux-bridges-and-vlans-in-unnamed-linux-network-namespaces-vi/)

## Preparation
* Make sure 802.1q module is loaded
```
[[ -z $(lsmod | grep 8021q) ]] &&  modprobe 8021q
```


## Creating the network namespaces, Linux bridges and the veth sub-interfaces
```
unshare --net --uts /bin/bash &
export pid_netns1=$!
nsenter -t $pid_netns1 -u hostname netns1
unshare --net --uts /bin/bash &
export pid_netns2=$!
unshare --net --uts /bin/bash &
export pid_netns3=$!
unshare --net --uts /bin/bash &
export pid_netns4=$!
unshare --net --uts /bin/bash &
export pid_netns5=$!
unshare --net --uts /bin/bash &
export pid_netns6=$!
unshare --net --uts /bin/bash &
export pid_netns7=$!
unshare --net --uts /bin/bash &
export pid_netns8=$!

# assign different hostnames
nsenter -t $pid_netns1 -u hostname netns1
nsenter -t $pid_netns2 -u hostname netns2
nsenter -t $pid_netns3 -u hostname netns3
nsenter -t $pid_netns4 -u hostname netns4
nsenter -t $pid_netns5 -u hostname netns5
nsenter -t $pid_netns6 -u hostname netns6
nsenter -t $pid_netns7 -u hostname netns7
nsenter -t $pid_netns8 -u hostname netns8

#set up veth devices in netns1 to netns4 with connection to netns3
ip link add veth11 netns $pid_netns1 type veth peer name veth13 netns $pid_netns3
ip link add veth22 netns $pid_netns2 type veth peer name veth23 netns $pid_netns3
ip link add veth44 netns $pid_netns4 type veth peer name veth43 netns $pid_netns3
ip link add veth55 netns $pid_netns5 type veth peer name veth53 netns $pid_netns3

#set up veth devices in netns6 and netns7 with connection to netns8
ip link add veth66 netns $pid_netns6 type veth peer name veth68 netns $pid_netns8
ip link add veth77 netns $pid_netns7 type veth peer name veth78 netns $pid_netns8

# Assign IP addresses and set the devices up
nsenter -t $pid_netns1 -u -n /bin/bash
ip addr add 192.168.5.1/24 brd 192.168.5.255 dev veth11
ip link set veth11 up
ip link set lo up
exit
nsenter -t $pid_netns2 -u -n /bin/bash
ip addr add 192.168.5.2/24 brd 192.168.5.255 dev veth22
ip link set veth22 up
ip link set lo up
exit
nsenter -t $pid_netns4 -u -n /bin/bash
ip addr add 192.168.5.4/24 brd 192.168.5.255 dev veth44
ip link set veth44 up
ip link set lo up
exit
nsenter -t $pid_netns5 -u -n /bin/bash
ip addr add 192.168.5.5/24 brd 192.168.5.255 dev veth55
ip link set veth55 up
ip link set lo up
exit
nsenter -t $pid_netns6 -u -n /bin/bash
ip addr add 192.168.5.6/24 brd 192.168.5.255 dev veth66
ip link set veth66 up
ip link set lo up
exit
nsenter -t $pid_netns7 -u -n /bin/bash
ip addr add 192.168.5.7/24 brd 192.168.5.255 dev veth77
ip link set veth77 up
ip link set lo up
exit

# set up bridge brx and its ports
nsenter -t $pid_netns3 -u -n /bin/bash
brctl addbr brx
ip link set brx up
ip link set veth13 up
ip link set veth23 up
ip link set veth43 up
ip link set veth53 up
brctl addif brx veth13
brctl addif brx veth23
brctl addif brx veth43
brctl addif brx veth53
exit

# set up bridge bry and its ports
nsenter -t $pid_netns8 -u -n /bin/bash
brctl addbr bry
ip link set bry up
ip link set veth68 up
ip link set veth78 up
brctl addif bry veth68
brctl addif bry veth78
exit
```

At this moment, `netns1-5` can ping each other, `netns6-7` can ping each other, but the two groups are separated by the two bridges, `brx` and `bry`.


## Set up the VLANs
```
# set up 2 VLANs on each bridge
nsenter -t $pid_netns3 -u -n /bin/bash
ip link set dev brx type bridge vlan_filtering 1
bridge vlan add vid 10 pvid untagged dev veth13
bridge vlan add vid 10 pvid untagged dev veth23
bridge vlan add vid 20 pvid untagged dev veth43
bridge vlan add vid 20 pvid untagged dev veth53
bridge vlan del vid 1 dev brx self
bridge vlan del vid 1 dev veth13
bridge vlan del vid 1 dev veth23
bridge vlan del vid 1 dev veth43
bridge vlan del vid 1 dev veth53
bridge vlan show
exit
nsenter -t $pid_netns8 -u -n /bin/bash
ip link set dev bry type bridge vlan_filtering 1
bridge vlan add vid 10 pvid untagged dev veth68
bridge vlan add vid 20 pvid untagged dev veth78
bridge vlan del vid 1 dev bry self
bridge vlan del vid 1 dev veth68
bridge vlan del vid 1 dev veth78
bridge vlan show
exit
```

At this moment, `netns1-2` can ping each other, `netns3-4` can ping each other; the other namespaces are not reachable because the isolation by vlan 10, and vlan 20.

## Create a veth device to connect the two bridges
```
ip link add vethx netns $pid_netns3 type veth peer name vethy netns $pid_netns8
nsenter -t $pid_netns3 -u -n /bin/bash
ip link add link vethx name vethx.50 type vlan id 50
ip link add link vethx name vethx.60 type vlan id 60
brctl addif brx vethx.50
brctl addif brx vethx.60
ip link set vethx up
ip link set vethx.50 up
ip link set vethx.60 up
bridge vlan add vid 10 pvid untagged dev vethx.50
bridge vlan add vid 20 pvid untagged dev vethx.60
bridge vlan del vid 1 dev vethx.50
bridge vlan del vid 1 dev vethx.60
bridge vlan show
exit

nsenter -t $pid_netns8 -u -n /bin/bash
ip link add link vethy name vethy.50 type vlan id 50
ip link add link vethy name vethy.60 type vlan id 60
brctl addif bry vethy.50
brctl addif bry vethy.60
ip link set vethy up
ip link set vethy.50 up
ip link set vethy.60 up
bridge vlan add vid 10 pvid untagged dev vethy.50
bridge vlan add vid 20 pvid untagged dev vethy.60
bridge vlan del vid 1 dev vethy.50
bridge vlan del vid 1 dev vethy.60
bridge vlan show
exit

nsenter -t $pid_netns8 -u -n tcpdump -n -i bry -e
nsenter -t $pid_netns3 -u -n tcpdump -n -i brx -e

nsenter -t $pid_netns7 -n ping 192.168.5.4
nsenter -t $pid_netns7 -n ping 192.168.5.1
```

## Two virtual VLANs spanning two Linux bridges connected by a veth based trunk line between trunk ports

```
# Change vethx to trunk like interface in brx
nsenter -t $pid_netns3 -u -n /bin/bash
brctl delif brx vethx.50
brctl delif brx vethx.60
ip link del dev vethx.50
ip link del dev vethx.60
brctl addif brx vethx
bridge vlan add vid 10 tagged dev vethx
bridge vlan add vid 20 tagged dev vethx
bridge vlan del vid 1 dev vethx
bridge vlan show
exit

# Change vethy to trunk like interface in brx
nsenter -t $pid_netns8 -u -n /bin/bash
brctl delif bry vethy.50
brctl delif bry vethy.60
ip link del dev vethy.50
ip link del dev vethy.60
brctl addif bry vethy
bridge vlan add vid 10 tagged dev vethy
bridge vlan add vid 20 tagged dev vethy
bridge vlan del vid 1 dev vethy
bridge vlan add vid 10 pvid untagged dev veth68
bridge vlan add vid 20 pvid untagged dev veth78
bridge vlan del vid 36 dev veth68
bridge vlan del vid 46 dev veth78
bridge vlan show
exit


```
