# Joey Rebuild #
Joey (ramone) is an edgy server. Joey has 3 main jobs. Stage web content to be pushed or synced to external servers. Provide disk replication for the lan file servers. Provide container space both for lan/wan/edge services and to back up those provided by the file server.
![network diagram](img/HomeNetworkDiagram.jpg)   
## Ubuntu 20.04 + zfs root on the Hp z400.
Someday this will not be so hard :)
As much as I like this little workhorse the bios on it kind of sucks. No UEFI. No booting from the on board raid. No booting from the external raid controller.

The current desktop installer can install zfs boot and root disks. It only works with UEFI based bios's but (much like the 18.04 install using the half baked on board psudo-raid controller) <em>it doesnt notice that your system doesnt support UEFI</em>.  It installs just fine but won't boot. So like the last install I just installed a minimal system and enough zfs tools to detect the installation and let grub find it. (then edit /etc/grub/default to boot to the zfs option and update-grub)

```
... booting from minimal ...
# nano /etc/default/grub
...
GRUB_DEFAULT=2
#GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="verbose"
GRUB_CMDLINE_LINUX="text"
# update-grub
# reboot
```

While your fixing things get rid of the graphical desktop.

```
... booting from zfs installation ...
root@joey:/# systemctl set-default multi-user.target

```

Then I added the mirror partitions. I didn't bother with the boot partitions on the mirror disk since they didn't work anyway. 

### at which point I realized I needed to make space for the containers.
Since the installer only accepted a disk name for the zfs install it took the entire disk. Fortunately we are running zfs. 
Booting from our minimal install we can break the mirrors we just created and then use the extra disks to recreate a smaller boot disk. (Note: root and boot pools are on sde and sdc).
#### shrinking a zfs pool 
``` 
zpool detach ata-TEAML5Lite3D240G_AB20190109A0101064-part6
zpool export rpool
zpool import rpool oldrpool

mkdir /oldroot /newroot
fdisk /dev/sde
.... delete partition six and split it into 2 new partitions ....
ls -lsa /dev/disk/by-id/|grep sde7
zpool create rpool ata-TEAML5Lite3D240G_AB20190109A0101064-part7
zpool export oldrpool
zpool import -R/oldroot oldrpool
zpool export rpool
zpool import -R/newroot rpool
zfs snapshot -r oldrpool@for_copy
zfs send -R oldrpool@for_copy | zfs recv -F rpool
zpool list
zpool export oldrpool
```

##### attach mirror to boot pool.
```
ls -ls /dev/disk/by-partuuid/|grep sde6
zpool attach bpool ata-TEAML5Lite3D240G_AB20190109A0101064-part6
zpool status bpool
zpool detach bpool d3bff208-06
update-grub
fdisk -l
```
##### copy partitions
```
sgdisk -p /dev/sde
sgdisk -R/dev/sdc /dev/sde
sgdisk -G /dev/sdc
```
##### 

### Disk Layout. 

* SSDs
* Archive disks

### Network Configuration 
[etc/netplan/50-cloud-init.yaml](etc/netplan/50-cloud-init.yaml)

* br0 is the isp router side of the network and provides an anonymous bridge.
* br1 is the internal network and is configured to provide direct connection to the server. Set the MTU of this system to 9000 for large transfers between servers.

### lxd setup.

```
... snap install lxd --channel=4.0/stable
root@joey:/home/don# apt-get install lxd
...
root@joey:/home/don# lxd init
Would you like to use LXD clustering? (yes/no) [default=no]: 
Do you want to configure a new storage pool? (yes/no) [default=yes]: 
Name of the new storage pool [default=default]: devil
Name of the storage backend to use (dir, lvm, zfs, ceph, btrfs) [default=zfs]: 
Would you like to create a new zfs dataset under rpool/lxd? (yes/no) [default=yes]: no
Create a new ZFS pool? (yes/no) [default=yes]: 
Would you like to use an existing empty block device (e.g. a disk or partition)? (yes/no) [default=no]: yes
Path to the existing block device: /dev/sdc2
Would you like to connect to a MAAS server? (yes/no) [default=no]: 
Would you like to create a new local network bridge? (yes/no) [default=yes]: no
Would you like to configure LXD to use an existing bridge or host interface? (yes/no) [default=no]: yes
Name of the existing bridge or host interface: br0
Would you like LXD to be available over the network? (yes/no) [default=no]: yes
Address to bind LXD to (not including port) [default=all]: 192.168.129.65
Port to bind LXD to [default=8443]: 
Trust password for new clients: 
Again: 
Would you like stale cached images to be updated automatically? (yes/no) [default=yes] 
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: yes
config:
  core.https_address: 192.168.129.65:8443
  core.trust_password: NOT_HERE
networks: []
storage_pools:
- config:
    source: /dev/disk/by-id/ata-TEAML5Lite3D240G_AB20190109A0101064-part2
  description: ""
  name: devil
  driver: zfs
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      nictype: bridged
      parent: br0
      type: nic
    root:
      path: /
      pool: devil
      type: disk
  name: default
cluster: null

```
### Restoring profiles and containers from lan file server.

```
root@annie:/home/don# lxc remote remove joey
root@annie:/home/don# lxc remote add joey
Certificate fingerprint: 2ad747d1305470bfd6787c1451e0ab64e22ab1798ae78d113718205763639742
ok (y/n)? yes
Admin password for joey: 
Client certificate stored at server:  joey
root@annie:/home/don# lxc profile copy infra joey:
root@annie:/home/don# lxc profile copy susdev20 joey:
root@annie:/home/don# lxc profile copy susdev21 joey:
root@annie:/home/don# lxc move nina joey:

```

### linkdump
* [https://openzfs.github.io/openzfs-docs/Getting%20Started/Ubuntu/Ubuntu%2020.04%20Root%20on%20ZFS.html#rescuing-using-a-live-cd](https://openzfs.github.io/openzfs-docs/Getting%20Started/Ubuntu/Ubuntu%2020.04%20Root%20on%20ZFS.html#rescuing-using-a-live-cd)
* [https://gist.github.com/yorickdowne/a2a330873b16ebf288d74e87d35bff3e](https://gist.github.com/yorickdowne/a2a330873b16ebf288d74e87d35bff3e)
* [https://saveriomiroddi.github.io/Installing-Ubuntu-on-a-ZFS-root-with-encryption-and-mirroring/#cloning-the-efi-partition](https://saveriomiroddi.github.io/Installing-Ubuntu-on-a-ZFS-root-with-encryption-and-mirroring/#cloning-the-efi-partition)
* [https://www.reddit.com/r/linuxadmin/comments/j8qzdq/install_ubuntu_server_2004_on_a_zfs_root/](https://www.reddit.com/r/linuxadmin/comments/j8qzdq/install_ubuntu_server_2004_on_a_zfs_root/)
* [https://www.medo64.com/2020/04/installing-uefi-zfs-root-on-ubuntu-20-04/](https://www.medo64.com/2020/04/installing-uefi-zfs-root-on-ubuntu-20-04/)

