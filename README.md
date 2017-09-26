# Arch installation on a BTRFS root filesystem
This is a cheatsheet with all the instructions to perform an installation of Arch Linux using BTRFS filesystem in /

[TOC]

## Laptop model ##
The installation has been done on a [Mountain Onyx](https://www.mountain.es/portatiles/onyx) laptop.

- **Screen**: 15,6" Full HD IPS Mate
- **CPU**: Intel® Core™ i7 6700HQ - 4C/8T
- **RAM**: 8GB DDR3L 1600 SODIMM
- **Hard Disk**: SSD 240GB M.2 550MB/s
- **GPU**: Nvidia GTX 960M 2GB GDDR5 + Intel i915 (Skylake)
- **UEFI**

## First steps ##
I have been following [Arch Wiki Beginner's Guide](https://wiki.archlinux.org/index.php/beginners'_guide). Arch ISO booted in UEFI mode using **systemd-boot**. It was configured a wired connection too.

## Partitioning ##
This is the selected layout for the UEFI/GPT system:

| Mount point | Partition | Partition type      | Bootable flag | Size   |
|-------------|-----------|---------------------|---------------|--------|
| /boot       | /dev/sda1 | EFI System Partition| Yes           | 512 MiB|
| [SWAP]      | /dev/sda2 | Linux swap          | No            | 16 GiB |
| /           | /dev/sda3 | Linux (BTRFS)       | No            | 30 GiB |
| /home       | /dev/sda4 | Linux (EXT4)        | No            | 192 GiB|

After creating the partitions, it was necessary to format them. Again, I followed the guide without a problem.

**IMPORTANT**: I used **-L** option with *mkfs* command in order to create a label for **/** (arch) and **/home** (home) partitions. It is important because fstab and the file needed to set up systemd-boot are configured to point those labels.

## BTRFS layout ##
The only partition formated with BTRFS was /dev/sda3, which contains the whole root system. BTRFS was selected to enable snapshots support in order to avoid any possible problem with Arch updates. Sometimes, I have experimented that certain critical package updates can break the system. If it occurs, it is a good idea to have some snapshot to rollback the entire root filesystem. tmp subvolume has been created in order to avoid the inclusion of temporal files within the snapshots of rootvol. tmp snapshots never will be created, because all the files stored within tmp are temporal. This is the layout defined:

```
sda3 (Volume)
|
|
- _active (Subvolume)
|    |
|    - rootvol (Subvolume - It will be the current /)
|    |
|    - tmp (Subvolume - It will be the current /tmp)
|
|
- _snapshots (Subvolume -  It will contain all the snapshots which are subvolumes too)
```
And these are the commands:

```
mkfs.btrfs -L arch /dev/sda3
mount /dev/sda3 /mnt
cd /mnt
btrfs subvolume create _active
btrfs subvolume create _active/rootvol
btrfs subvolume create _active/tmp
btrfs subvolume create _snapshots
```

Next, mount all the partitions (/boot included) in order to start the installation:

```
cd
umount /mnt
mount -o subvol=_active/rootvol /dev/sda3 /mnt
mkdir /mnt/{home,tmp,boot}
mount -o subvol=_active/tmp /dev/sda3 /mnt/tmp
mount /dev/sda1 /mnt/boot
mount /dev/sda4 /mnt/home
```

## Installing Arch Linux ##
Proceed with installing Arch Linux (Installation section within Beginner's guide).

## Fstab ##
After executing *genfstab -U /mnt >> /mnt/etc/fstab* to generate fstab file using **UUIDs** for the partitions, I edited fstab and this is the result (please note that for those partitions which have a label defined, this label has been used)

```
#
# /etc/fstab: static file system information
#
# <file system> <dir>   <type>  <options>       <dump>  <pass>
# /dev/sda3
LABEL=arch      /               btrfs           rw,relatime,compress=lzo,ssd,discard,autodefrag,space_cache,subvol=/_active/rootvol     0 0

# /dev/sda1
UUID=C679-F6A0          /boot           vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro    0 2

# /dev/sda3
LABEL=arch      /tmp            btrfs           rw,relatime,compress=lzo,ssd,discard,autodefrag,space_cache,subvol=/_active/tmp 0 0

# /dev/sda4
LABEL=home       /home           ext4            rw,relatime,data=ordered        0 2

# /dev/sda2
UUID=04293b56-e2f9-4d3b-aded-6baad666d5bb       none            swap            defaults        0 0

# /dev/sda3 LABEL=arch volume
LABEL=arch      /mnt/defvol             btrfs           rw,relatime,compress=lzo,ssd,discard,autodefrag,space_cache     0 0

```
最后一行 ，不知道什么意思 /mnt/defvol  是什么？
如果挂载时不指定 subvol= 选项便会挂载默认子卷.要改变默认子卷:
在安装了 GRUB 的系统上,在改变默认子卷以后不要忘记运行 grub-install .
grub-install --target=i386-pc --recheck /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg

## Mkinitcpio ##
In order to enable BTRFS on initramfs image, I added **btrfs** on HOOK inside **/etc/mkinitcpio.conf**. Then, it was necessary to execute **mkinitcpio -p linux** again. If you install linux-lts kernel (Long Term Support), you will have to execute **mkinitcpio -p linux-lts**
这个加了 btrfs了，有的说不用加， 上次我是加了的，没事儿。
fsck有人说要改成brtfsck，，后来想了一下不用，改成btrfsck是因为整盘都是btrfs格式的，
我这是混合的，将来恢复也主要是恢复ext4格式的，所以不改fsck



## Bootloader ##
Because this is a UEFI laptop, [systemd-boot](https://wiki.archlinux.org/index.php/Systemd-boot) was used as a **bootloader**. First, it was necessary to install systemd-boot. /boot partition (/dev/sda1) was previously mounted, so it was only needed to execute **bootctl install**.
Then, I edited **/boot/loader/loader.conf**. This is the final content of this file:

```
timeout 3
default arch-btrfs
editor 0
```
**bootctl install**.  不知道什么意思


Then, I installed **intel-ucode** package because I have an Intel CPU as beginners guide says.

Please, also note that **arch-btrfs** (above configuration) is the file created within **/boot/loader/entries** and its name is **arch-btrfs.conf**. This is the boot entry that we need to see Arch Linux option when the laptop boots, and the content of the file is:

```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=LABEL=arch rw rootflags=subvol=_active/rootvol
```

The line **initrd  /intel-ucode.img** enables Intel microcode updates (installed with intel-ucode package)

Please note that the line **options root=LABEL=arch rw rootflags=subvol=_active/rootvol** assumes that root partition is installed in the labeled as arch, so here it is no necessary to use PUUID, UUID or the name of the partition, only the label (using LABEL variable).

## Additional packages installed ##
A bunch of useful packages has been installed too: [tlp](https://wiki.archlinux.org/index.php/TLP) for energy saving and advanced power management, [reflector](https://wiki.archlinux.org/index.php/Reflector) for optimizing Arch mirrors repositories, [yaourt](http://www.ostechnix.com/install-yaourt-arch-linux/) for compiling and installing packages easily from AUR repository, [snapd](https://wiki.archlinux.org/index.php/Snapd) to install snap packages, [btrfs-progs](https://wiki.archlinux.org/index.php/Btrfs) to manage BTRFS filesystem.

## Finish the steps in the Wiki ##
And reboot!!

## Automated snapshots and system updates ##
A very simple script has been created called **upgrade-system.sh** and stored within **/usr/bin/upgrade-system.sh** for system upgrades. Before starting the installation of the packages updated, the script creates a new snapshot. Then, pacman, yaourt and snapd are launched to upgrade the system. If something went wrong, you can restore the root filesystem using the last snapshot created. This script can be found [here](https://github.com/egara/arch-btrfs-installation/blob/master/files/upgrade-system.sh)

## Configuration files ##
In this [folder](https://github.com/egara/arch-btrfs-installation/tree/master/files) you can find all the configuration files edited or created during the installation process.

## Re-installing the system ##
The previous way didn't work as I expected. Because of /boot partition is independent, if you want to rollback to a previous snapshot with a different kernel installed there is a problem. I don't snapshot /boot, so there it is always the images generated for the last kernel installed. This is a problem! So I reinstalled the whole system disabling UEFI mode and enabling legacy BIOS. Then, I partitioned the system using only thre partitions: sda1 for / (inlcuidng boot partition), sda2 for swap and sda3 for home. sda1 is BTRFS, but because of the whole root system is stored there, now when I snapshot this partition, /boot is included and there is no problem with different kernel installations. I used GRUB as a boot loader.


## Problem with Docker and BTRFS ##
More than a problem is a caveat. If the main filesystem  for root is BTRFS, docker will use BTRFS storage driver (Docker selects the storage driver automatically depending on the system's configuration when it is installed) to create and manage all the docker images, layers and volumes. It is ok, but there is a problem with snapshots. Because **/var/lib/docker** is created to store all this stuff in a BTRFS subvolume which is into root subvolume, all this data won't be included within the snapshots. In order to allow all this data be part of the snapshots, we will change the storage driver used by Docker. It will be used **devicemapper**. Please, check out [this reference](https://docs.docker.com/engine/userguide/storagedriver/selectadriver/) in order to select the proper storage driver for you. You must know that depending on the filesystem you have for root, some of the storage drivers will not be allowed.

For using devicemapper:
- Install docker
- Create a file called **storage-driver.conf** within **/etc/systemd/system/docker.service.d/**. If the directory downs't exist, create the directory first.
- This is the content of **storage-driver.conf**
```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// --storage-driver=devicemapper
```

- Create **/var/lib/docker/** and disable CoW (copy on write for BTRFS):
```
sudo chattr +C /var/lib/docker
```

- Enable and start the service
```
sudo systemctl enable docker.service
sudo systemctl start docker.service
```

- Add your user to docker group in order to use docker command withou sudo superpowers!

## Other tips ##

- I have installed [Antergos](https://antergos.com/) (Arch-based distro easy to install and to go without too much configuration) on a PC that I needed to work inmediately. I used BTRFS too for the installation, but the problem is that you cannot choose the layout you want for your BTRFS volume. Instead, all the root system installed directly in the top volume itself, but I want a more refined layout (the layout defined above) in order to manage all the snapshots in a more proper way. Because of that, I detailed all the steps I made in order to mmigrate my installation to a customize layout.
Once the system is installed, reboot and open a terminal to see the structure of the BTRFS volume for /:
```
sudo btrfs subvolume list /
```
Create all the subvolumes on / except rootvol subvolume:
```
btrfs subvolume create _active
btrfs subvolume create _active/tmp
btrfs subvolume create _snapshots
```
Now, it is necessary to make a read-write snapshot of / into _active/rootvol
```
sudo btrfs subvolume snapshot / /_active/rootvol
```
到这儿是不是应该把/ 下面的文件 rm -rf 啊  rm是没法删除subvol的，安全。
Modify fstab to reflect the changes (remember to modify / entry and point it to /_active/rootvol. Add /tmp line too). it is interesting to create a new directory within /mnt/defvol in order to mount the entire volume as it is described above too.
Reboot the system using Archlinux LiveCD or Antergos LiveCD.
Once the system is booted, mount all the structure within /mnt using as root /_active/rootvol (in my case, / is in /dev/sda1 and /home is in /dev/sdb2):
```
mount -o subvol=/_active/rootvol /dev/sda1 /mnt
mount -o subvol=/_active/tmp /dev/sda1 /mnt/tmp
mount /dev/sdb2 /mnt/home
```
还有其它的  swap
其实没必要弄这种两层结构吧，用一层    /root  /tmp /snapshot 就行吧。  两层有什么意义呢
Chroot the new system:
```
arch-chroot /mnt /bin/bash
```
Add **btrfs** as HOOK within /etc/mkinitcpio.conf and rebuild images:
```
mkinitcpio -p linux
```
Reinstall GRUB (in my case, the PC was installed in BIOS legacy mode and GRUB is installed on /dev/sda):
```
grub-install --target=i386-pc /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```
Exit chroot and unmount everything.
```
exit
umount /mnt -R
```
Reboot.
Once the system is booted, check if / is pointing to /_active/rootvol. if everything is working fine, all the files within the root of the volume can be deleted using rm -rf boot bla bla bla. If systemd created the subvolume /var/lib/machines in the root of the volume, don't delete it and add it to fstab too.
This is the [original post](http://unix.stackexchange.com/questions/62802/move-a-linux-instalation-using-btrfs-on-the-default-subvolume-subvolid-0-to-an) from I got the inspiration to do this stuff.


有快照后怎么恢复呢？ 看网上说可以利用snapshot和 default subvolume ，我还没有搞明白。




subvolume不能用rm来删除，只能通过btrfs命令来删除。默认情况下subvolume的快照是可写的
快照是特殊的subvolume，具有subvolume的属性。所以快照也可以通过mount挂载，也可以通过btrfs property命令设置只读属性
由于快照的本质就是一个subvolume，所以可以在快照上面再做快照
#在做一个subvolume的快照的时候，不会将它里面的subvolume也做快照https://segmentfault.com/a/1190000008605135
