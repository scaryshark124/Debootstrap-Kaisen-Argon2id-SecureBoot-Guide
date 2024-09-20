# Build Kaisen Linux using debootstrap with Argon2id and Secure Boot Guide
I will demonstrate how to use debootstrap to build Kaisen Linux on a USB thumb drive. While in chroot, I will demonstrate how to compile grub2 with ArchLinux's grub-improved-luks2 that allows me to encrypt the root system with argon2id password hashing algorithm. Lastly, I will demonstrate how to create machine owner key (MOK) to sign grub bootloader and vmlinuz.

### What you need:
  Debian live ISO (12.5.0 or later)
     - https://www.debian.org/CD/live/
  
  USB 3.0 thumb drives with at least 32GB

## Build Environment
In order to follow along with this guide you will need Debian. Although, you can use this guide (with a few tweaks) with other linux distributions. I will be using debian-live-12.5.0-amd64-xfce.iso to make the commands in this guide flow more smoothly if you build from a fresh Debian live Iso. However, with that said, you should have a basic understanding of linux commands and file structure to troubleshoot. If you are new to Linux I will suggest a free Introduction to Linux course (https://training.linuxfoundation.org/training/introduction-to-linux/) from the Linux Foundation.

Here I will open a terminal and login as superuser. I will create a variable for the mount point. This is an important step as the commands in this guide will call this variable. 
```
user@debian:~$ sudo su -

root@debian:~# mkdir -vp /mnt/kaisen

root@debian:~# echo 'CB="/mnt/kaisen"' >> ~/.bashrc

root@debian:~# source ~/.bashrc

root@debian:~# echo $CB
/mnt/kaisen
```
## Sections

   1. Installing Dependencies
 
   2. Partitioning the USB thumb drive
   
   3. Creating the Luks2 encrypted partition
   
   4. Creating Physical volume, Volume group, Logical Volume
   
   5. Creating the File systems
   
   6. Mounting the root and EFI partitions
   
   7. Prepare debootstrap to build Kaisen
   
   8. Debootstrap Kaisen
   
   9. Prepare for chroot
   
   10. chroot into Kaisen
       - Build the system
       - Install base packages
       - Install build dependencies for Grub
       - Create key for initramfs unlock
       - Link key to unlock partition during initramfs stage
       - Create user accounts
       - Prepare for building grub2 from source
       - Build grub2 including ArchLinux's argon2 patches
       - Create grub configuration and install grub2 to EFI
       - Install signed shim and MOK key manager to EFI
       - Create Machine Owner Key (MOK)
       - Sign and verify vmlinuz and grubx64.efi
       - Set boot entry for signed shim
  
  11. Exit chroot, close logical volumes, and physical volume.
  12. Reboot and register MOK with MOK manager.

## Installing Dependencies

I will edit the sources.list to include contrib and non-free. Then I will go ahead update and upgrade.  

```
sed 's|deb http://deb.debian.org/debian/ bookworm main non-free-firmware|deb http://deb.debian.org/debian/ bookworm main contrib non-free non-free-firmware|g' -i "/etc/apt/sources.list"

root@debian:~# apt update && apt upgrade
```
I will install debootstrap and arch-install-scripts. I am arch-install-scripts because it is quicker to get in and out of chroot, and genfstab cuts out the hassle of manually configuring /etc/fstab. 
```
root@debian:~# apt install arch-install-scripts debootstrap -y
```

## Partitioning the USB thumb drive

you can use commands such as 'lsusb' or 'dmesg' to identify the USB you plan to build on, but it is much simpler to run 'lsblk' before you plug in the USB, and then again after you plug in the USB. I will be using a 32GB USB, which on my system is '/dev/sdb'
```
root@debian:~# lsblk
NAME                 MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
loop0                  7:0    0   2.5G  1 loop  /usr/lib/live/mount/rootfs/filesystem.squashfs
                                                /run/live/rootfs/filesystem.squashfs
sda                    8:0    1   239G  0 disk  
├─sda1                 8:1    1   500M  0 part  
│ └─cryptlocker      253:0    0   484M  0 crypt 
│   └─G1-CryptLocker 253:1    0   480M  0 lvm   /mnt
└─sda2                 8:2    1 238.5G  0 part  /usr/lib/live/mount/medium
                                                /run/live/medium
sdb                    8:16   1  28.7G  0 disk  
├─sdb1                 8:17   1   200M  0 part  
└─sdb2                 8:18   1  28.5G  0 part  
```

When creating a new bootable USB I will usually do at least a two pass wipe.

This fills the drive with a urandom (unlimited random) stream of data from the entropy pool, and then from algorithims such as SHA or MD5 when entropy pool is empty:
```
root@debian:~# dd if=/dev/urandom of=/dev/sdb bs=4096 status=progress
```
This fills the drive with zeros:
```
root@debian:~# dd if=/dev/zero of=/dev/sdb bs=4096 status=progress
```
I will use 'gdisk' to partition the USB:
```
root@debian:~# gdisk /dev/sdb
GPT fdisk (gdisk) version 1.0.9

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries in memory.

Command (? for help): o
This option deletes all partitions and creates a new protective MBR.
Proceed? (Y/N): Y

Command (? for help): n
Partition number (1-128, default 1): 
First sector (34-60088286, default = 2048) or {+-}size{KMGTP}: 
Last sector (2048-60088286, default = 60086271) or {+-}size{KMGTP}: +200M
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300): ef00
Changed type of partition to 'EFI system partition'

Command (? for help): n
Partition number (2-128, default 2): 
First sector (34-60088286, default = 411648) or {+-}size{KMGTP}: 
Last sector (411648-60088286, default = 60086271) or {+-}size{KMGTP}: 
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/sdb: 60088320 sectors, 28.7 GiB
Model: Cruzer Glide    
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): AE5CCD76-96D9-40E1-85A7-B1B882300974
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 60088286
Partitions will be aligned on 2048-sector boundaries
Total free space is 4029 sectors (2.0 MiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048          411647   200.0 MiB   EF00  EFI system partition
   2          411648        60086271   28.5 GiB    8300  Linux filesystem

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/sdb.
The operation has completed successfully.
```

## Creating the Luks2 encrypted partition

I will be using luks2 with argon2id:
```
root@debian:~# cryptsetup luksFormat --pbkdf=argon2id --use-urandom -s 512 -h sha512 -i 10000 /dev/sdb2

WARNING!
========
This will overwrite data on /dev/sdb2 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sdb2: 
Verify passphrase:
```
I will open the newly created encrypted partition:
```
root@debian:~# cryptsetup open /dev/sdb2 kaisen-cryptlvm
Enter passphrase for /dev/sdb2:
```
 
## Creating Physical volume, Volume group, Logical Volume

I will create the physical volume, volume group, and logical volumes for the swap and root partition:
```
root@debian:~# pvcreate /dev/mapper/kaisen-cryptlvm
  Physical volume "/dev/mapper/kaisen-cryptlvm" successfully created.

root@debian:~# vgcreate vg1 /dev/mapper/kaisen-cryptlvm
  Volume group "vg1" successfully created
 		
root@debian:~# lvcreate -L 4G vg1 -n swap
  Logical volume "swap" created.
		
root@debian:~# lvcreate -l +100%FREE vg1 -n root
  Logical volume "root" created.

root@debian:~# lsblk
NAME                 MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
loop0                  7:0    0   2.5G  1 loop  /usr/lib/live/mount/rootfs/filesystem.squashfs
                                                /run/live/rootfs/filesystem.squashfs
sda                    8:0    1   239G  0 disk  
├─sda1                 8:1    1   500M  0 part  
│ └─cryptlocker      253:0    0   484M  0 crypt 
│   └─G1-CryptLocker 253:1    0   480M  0 lvm   /mnt
└─sda2                 8:2    1 238.5G  0 part  /usr/lib/live/mount/medium
                                                /run/live/medium
sdb                    8:16   1  28.7G  0 disk  
├─sdb1                 8:17   1   200M  0 part  
└─sdb2                 8:18   1  28.5G  0 part  
  └─kaisen-cryptlvm  253:2    0  28.4G  0 crypt 
    ├─vg1-swap       253:3    0     4G  0 lvm   
    └─vg1-root       253:4    0  24.4G  0 lvm  
```

## Creating the File systems

I will create the filesystems for the EFI, swap, and root partitions. 
```
root@debian:~# mkfs.fat -F 32 /dev/sdb1
mkfs.fat 4.2 (2021-01-31)
 

root@debian:~# mkswap -L swap /dev/vg1/swap
Setting up swapspace version 1, size = 4 GiB (4294963200 bytes)
LABEL=swap, UUID=41338ea0-945a-4310-8291-818a3ea91b92


root@debian:~# mkfs.ext4 -L 'root' /dev/vg1/root
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 6406144 4k blocks and 1602496 inodes
Filesystem UUID: 61364c3c-c93c-4129-9692-50cfaac00e9a
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
	4096000

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done 
```

## Mounting the root and EFI partitions

I will mount the newly created filesystems to the host in preparation to build. 

As a sanity check:
```
root@debian:~# echo $CB
/mnt/kaisen
```

now create and mount:
```
root@debian:~# mkdir -vp $CB

root@debian:~# mount -v /dev/vg1/root $CB
mount: /dev/mapper/vg1-root mounted on /mnt/kaisen.


root@debian:~# mkdir -vp $CB/efi
mkdir: created directory '/mnt/kaisen/efi'

root@debian:~# mount -v /dev/sdb1 $CB/efi
mount: /dev/sdb1 mounted on /mnt/kaisen/efi.
```
I will turn on the swap for the build.
```
root@debian:~# swapon /dev/mapper/vg1-swap
```
If everything went alright, you should a similar output as below running 'lsblk':
```
root@debian:~# lsblk
NAME                 MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
loop0                  7:0    0   2.5G  1 loop  /usr/lib/live/mount/rootfs/filesystem.squashfs
                                                /run/live/rootfs/filesystem.squashfs
sda                    8:0    1   239G  0 disk  
├─sda1                 8:1    1   500M  0 part  
│ └─cryptlocker      253:0    0   484M  0 crypt 
│   └─G1-CryptLocker 253:1    0   480M  0 lvm   /mnt
└─sda2                 8:2    1 238.5G  0 part  /usr/lib/live/mount/medium
                                                /run/live/medium
sdb                    8:16   1  28.7G  0 disk  
├─sdb1                 8:17   1   200M  0 part  /mnt/kaisen/efi
└─sdb2                 8:18   1  28.5G  0 part  
  └─kaisen-cryptlvm  253:2    0  28.4G  0 crypt 
    ├─vg1-swap       253:3    0     4G  0 lvm   [SWAP]
    └─vg1-root       253:4    0  24.4G  0 lvm   /mnt/kaisen
```
## Prepare debootstrap to build Kaisen

Debootstrap uses scripts located in /usr/share/debootstrap/scripts. You will notice that a lot of scripts are generally the same, as most point to a Debian-common script. The differences between the scripts being keyring and default_mirror. I will create a kaisen-rolling script using the testing script as a template. This is in case the kaisen-rolling script doesn't already exist. If you have a kaisen rolling script, then you can skip this step. However, I would check to make everything looks like the one I create further into the guide. 
```
root@debian:~# cp /usr/share/debootstrap/scripts/testing /usr/share/debootstrap/scripts/kaisen-rolling
```
I will edit the kaisen-rolling script to point to the kaisen-archive-keyring.gpg that I will install next. I also added a default_mirror, but I usually include the mirror anyhow during build time. 
```
root@debian:~# sed 's|keyring /usr/share/keyrings/debian-archive-keyring.gpg|keyring /usr/share/keyrings/kaisen-archive-keyring.gpg|g' -i "/usr/share/debootstrap/scripts/kaisen-rolling"

root@debian:~# sed '6 idefault_mirror https://deb.kaisenlinux.org' -i "/usr/share/debootstrap/scripts/kaisen-rolling"
```
EXAMPLE MODIFIED TESTING SCRIPT:
```
mirror_style release
download_style apt
finddebs_style from-indices
variants - buildd fakechroot minbase
keyring /usr/share/keyrings/kaisen-archive-keyring.gpg
default_mirror https://deb.kaisenlinux.org

# include common settings
if [ -e "$DEBOOTSTRAP_DIR/scripts/debian-common" ]; then
 . "$DEBOOTSTRAP_DIR/scripts/debian-common"
elif [ -e /debootstrap/debian-common ]; then
 . /debootstrap/debian-common
elif [ -e "$DEBOOTSTRAP_DIR/debian-common" ]; then
 . "$DEBOOTSTRAP_DIR/debian-common"
else
 error 1 NOCOMMON "File not found: debian-common"
fi
```
You can use debootstrap to build without an archive keyring as it will just complain about it, and then build it anyway. However, I will include the kaisen-archive-keyring. I will download the kaisen-archive-keyring from their repository and install it.

You may have to find an updated link to the kaisen-archive-keyring. Just go through kaisen's repository or mirrors to select it. You can go through debian repositories easily using firefox to open https://deb.kaisenlinux.org/pool/main/k/. This will give you insight on how debian repositories are structured.

```
root@debian:~# wget https://deb.kaisenlinux.org/pool/main/k/kaisen-archive-keyring/kaisen-archive-keyring_2024+kaisen2_all.deb

root@debian:~# dpkg -i kaisen-archive-keyring_2024+kaisen2_all.deb
```

## Debootstrap Kaisen
If everything has went well so far you should be able to run this:
```
root@debian:~# debootstrap --components=main,contrib,non-free,non-free-firmware kaisen-rolling $CB https://deb.kaisenlinux.org
```

## Prepare for chroot
I will create a few things before using chroot. I will use a tool from arch-install-scripts to generate an fstab:
```
root@debian:~# genfstab -U $CB >> $CB/etc/fstab
```

I will create the hostname and hosts files:
```
root@debian:~# echo 'crunchy-kaisen' > $CB/etc/hostname

root@debian:~# cat > $CB/etc/hosts << EOF
127.0.0.1	localhost
127.0.1.1	crunchy-kaisen
::1  		localhost ip6-localhost ip6-loopback
ff02::1  	ip6-allnodes
ff02::2  	ip6-allrouters
EOF
```

## chroot into Kaisen
I will use arch-chroot from arch-install-scripts. This will automatically mount the psuedo filesystems and the resolv.conf for internet connection. arch-chroot will also automatically unmount when exiting chroot. This streamlines the chroot process.
```
root@debian:~# arch-chroot /$CB /bin/bash --login
```
  ### Build the system
I will set a new passowrd for root.
```
root@debian:/# passwd
New password: 
Retype new password: 
passwd: password updated successfully
```

I will install a locales and reconfigure tzdata. for locales, I will use en_US.UTF-8, this will need to be changed based on your location. I reconfigured my tzdata to reflect my own timezone, this may be different as it is based on your location. 
```
root@debian:/# apt update && apt upgrade

root@debian:/# apt install locales -y

root@debian:/# dpkg-reconfigure locales
Locales to be generated: 97
Default locale for the system environment: 2
Generating locales (this might take a while)...
  en_US.UTF-8... done
Generation complete.

root@debian:/# dpkg-reconfigure tzdata
Geographic area: 2
Time zone: 37
```  
I will install tools needed for initramfs and cryptsetup so that I can create /etc/crypttab for the encrypted root partition. 
```
root@debian:/# apt install initramfs-tools cryptsetup cryptsetup-initramfs -y
```
I will create the resume file in initramfs-tools conf.d directory
```
root@debian:/# grep '[[:blank:]]swap' /etc/fstab | grep UU | cut -c '6-41'
41338ea0-945a-4310-8291-818a3ea91b92

root@debian:/# > /etc/initramfs-tools/conf.d/resume printf 'RESUME=%s\n' "$(grep '[[:blank:]]swap' /etc/fstab | grep UU | cut -c '-41')"

root@debian:/# cat /etc/initramfs-tools/conf.d/resume
RESUME=UUID=41338ea0-945a-4310-8291-818a3ea91b92
```
I will install the linux headers and linux image along with the logical volume manager
```
root@debian:/# apt install --fix-missing lvm2 linux-headers-amd64 linux-image-amd64
```
 ### Install build dependencies for Grub
 
 I will install what I need to compile grub from source:
```
root@debian:/# apt install --fix-missing shim-signed shim-helpers-amd64-signed libalpm13t64 sudo git curl libarchive-tools help2man python3 rsync texinfo texinfo-lib ttf-bitstream-vera build-essential dosfstools efibootmgr uuid-runtime efivar mtools os-prober dmeventd libdevmapper-dev libdevmapper-event1.02.1 libdevmapper1.02.1 libfont-freetype-perl python3-freetype libghc-gi-freetype2-dev libghc-gi-freetype2-prof fuse2fs libconfuse2 libfuse2t64 gettext xorriso libisoburn1t64 libisoburn-dev autogen gnulib libfreetype-dev pkg-config m4 libtool automake flex fuse3 libfuse3-dev gawk autoconf-archive rdfind fonts-dejavu lzma lzma-dev liblzma5 liblzma-dev liblz1 liblz-dev unifont acl libzfslinux-dev sbsigntool
```  
  ### Create key for initramfs unlock
  root@debian:/# mkdir -vp /etc/keys
mkdir: created directory '/etc/keys'

Now we will make sure the luk2 volume gets unlocked during boot. Here is my take on including a key to unlock the root partition during the initramfs stage. So, during the boot process grub runs 'cryptomount -u SOME-UUID', this is the first prompt for password for grub to access the /boot directory. I want to note, that during this time the entire partition is unlocked, not just the /boot directory. So, if someone were able to get by the first password prompt and decyrpt the luks2 partition, then having the key is useless. As they will already have control over the entire luks2 partition, not just the /boot directory. I will take measures to ensure proper handling of the key during the initramfs stage. 

root@debian:/# ( umask 0077 && dd if=/dev/urandom bs=1 count=128 of=/etc/keys/root.key conv=excl,fsync )
128+0 records in
128+0 records out
128 bytes copied, 0.0195378 s, 6.6 kB/s

root@debian:/# chown -vR root:root /etc/keys
ownership of '/etc/keys/root.key' retained as root:root
ownership of '/etc/keys' retained as root:root

root@debian:/# chmod -vR 600 /etc/keys
mode of '/etc/keys' changed from 0755 (rwxr-xr-x) to 0600 (rw-------)
mode of '/etc/keys/root.key' retained as 0600 (rw-------)

root@debian:/# chattr +i /etc/keys/root.key
  
  ### Link key to unlock partition during initramfs stage

  After you move the keys and cert over then run these commands to make sure to set owner and group to root and change permissions so that only the owner can read, write, and execute. I will also set immutable. 

root@debian:/# cryptsetup --cipher aes-xts-plain64:sha512 -s 512 -h sha512 -i 10000 --pbkdf=argon2id luksAddKey /dev/sdb2 /etc/keys/root.key
Enter any existing passphrase: 

## I will now configure /etc/crypttab.

root@debian:/# echo "kaisen-cryptlvm UUID=$(blkid -s UUID -o value /dev/sdb2) /etc/keys/root.key luks,discard,key-slot=1" >> /etc/crypttab

root@debian:/# cat /etc/crypttab
# <target name>	<source device>		<key file>	<options>
kaisen-cryptlvm UUID=eb6fbac7-772d-43e7-bc7f-c67155c3b658 /etc/keys/root.key luks,discard,key-slot=1

I will add this line so initramfs is able to find the key

root@debian:/# echo "KEYFILE_PATTERN=\"/etc/keys/*.key\"" >>/etc/cryptsetup-initramfs/conf-hook

I will set UMASK to restrictive value to avoid leaking key material. I will then make sure restrictive permissions are set and the key is available in initramfs.

root@debian:/# echo UMASK=0077 >>/etc/initramfs-tools/initramfs.conf
  
  ### Create user accounts
  
  ### Prepare for building grub2 from source
  
  ### Build grub2 including ArchLinux's argon2 patches
  
  ### Create grub configuration and install grub2 to EFI
  
  ### Install signed shim and MOK key manager to EFI
  
  ### Create Machine Owner Key (MOK)
  
  ### Sign and verify vmlinuz and grubx64.efi
  
  ### Set boot entry for signed shim

## Exit chroot, close logical volumes, and physical volume

## Reboot and register MOK with MOK manager
                
