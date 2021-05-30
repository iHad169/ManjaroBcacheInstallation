## Manjaro Bcache Installation
Installation of Manjaro on Bcache Root Partition. Make Manjaro have SSD speed in HDD. 


### Partition Disks

```
=============================================================================================================
| /dev/sda (SSD)                                                        | /dev/sdb (HDD)                    |
=============================================================================================================
| /dev/sda1 (FAT32 EFI partition (if you are UEFI) For /boot/efi 300MB) | /dev/sdb1 (BCACHE backing device) |
| /dev/sda2 (EXT4 Boot partition For /boot 2GB)                         |                                   |
| /dev/sda3 (SWAP)                                                      |                                   |
| /dev/sda4 (BCACHE cache device)                                       |                                   |
=============================================================================================================
```
In my case the Partition Disks is below after successful installation.
```
$ lsblk
sda                         8:0    0 223.6G  0 disk 
├─sda1                      8:1    0   300M  0 part /boot/efi
├─sda2                      8:2    0     2G  0 part /boot
├─sda3                      8:3    0  17.2G  0 part [SWAP]
└─sda4                      8:4    0 204.1G  0 part 
  └─bcache0               254:0    0   3.6T  0 disk 
    └─VolumeGroup00-root  253:1    0   3.6T  0 lvm  /
sdb                         8:16   0   3.6T  0 disk 
└─sdb1                      8:18   0   3.6T  0 part 
  └─bcache0               254:0    0   3.6T  0 disk 
    └─VolumeGroup00-root  253:1    0   3.6T  0 lvm  /
```
### Boot Manjaro Linux Live CD USB
The `Manjaro Linux Live CD .iso` you can on [this download](https://manjaro.org/download/). After you burn the `.iso` to your USB device for boot to install Manjaro Linux. 
### Install Bcache Tools
First, connect to the Internet. Make sure the connection is working. We will install `bcache-tools` and create the `bcache` device.
```
$ sudo pacman -Sy --needed yay base-devel
$ yay -S bcache-tools
```
### Use Root Permission
Convenient for the next work.
```
$ sudo -i
```
### Create Bcache Partition
Wipe the cache and backing partition file systems.
```
# wipefs -a /dev/sdb1
# wipefs -a /dev/sda4
# make-bcache -B /dev/sdb1 -C /dev/sda4
```
Notice the command to `make-bcache` used the HDD partition, `/dev/sdb1`, as the backing (`-B`) device and the SDD partition, `/dev/sda4`, as the cache (`-C`) device. [Detail](https://wiki.archlinux.org/title/bcache)
### Create LVM Partition
Create LVM Partition For `Manjaro Linux Installer` install able. Because the `Manjaro Linux Installer` cannot find bcache device.
```
# pvcreate /dev/bcache0
# vgcreate VolumeGroup00 /dev/bcache0
# lvcreate -n root -l 100%FREE VolumeGroup00
```
### Create Btrfs Partition 
Add `Filesystem (Btrfs)` on LVM Partition for install `Manjaro`
```
# mkfs.btrfs /dev/VolumeGroup00/root
```
### Run Manjaro Linux Installer
Run the Manjaro Linux Installer until the installation is complete. **Remember do not restart your Manjaro Linux Live CD.** You need do some thing to boot
able the New Installation Manjaro.
### Chroot New Installation
Here is where things get tricky. What we’re going to do is switch to the new operating system without booting and install some software to get `bcache-tools `installed and a new `mkinitcpio` generated so the computer will boot.

First we are going to create a valid `manjaro-chroot` environment. We start by mounting several directories from the new installation into specific sub-directories in order to create the directory structure Manjaro Linux expects:
```
# mount /dev/VolumeGroup00/root -o subvol=@ /mnt
# mount /dev/sda2 /mnt/boot
# mount /dev/sda1 /mnt/boot/efi/
# manjaro-chroot /mnt /bin/bash
```
### Install Bcache Tools on New Installation
Now we are effectively within the new installation’s file system. So all we need to do is install `bcache-tools`.
```
# sudo pacman -Sy --needed yay base-devel
```
Please change `su <your_user_name>`. Install `bcache-tools` we need to non Root Permission. (In my case non Root Permission user is `localhost` )
```
# su localhost
$ yay -S bcache-tools
$ exit
```
### Update mkinitcpio
Edit `/etc/mkinitcpio.conf`
```
# gedit /etc/mkinitcpio.conf
```
Add `bcache` to the list of modules.
```
...
MODULES="bcache"
...
```
Also add `bcache lvm2` in that order to the list of hooks before `filesystems` but after `block`. (WARNING: The exact placement here is critical!) This is so linux knows how to read the BCache and LVM partitions.
```
...
HOOKS="... block bcache lvm2 filesystems ..."
...
```
Regenerate the Linux image in `/boot` If successful there will be no errors. (A few warnings is likely fine.)
```
# mkinitcpio -P
```
### Restart to new Installation Manjaro
Happy to use ^^
