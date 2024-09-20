# Build Kaisen Linux using debootstrap with Argon2id and Secure Boot Guide
I will demonstrate how to use debootstrap to build Kaisen Linux on a USB thumb drive. While in chroot, I will demonstrate how to compile grub2 with ArchLinux's grub-improved-luks2 that allows me to encrypt the root system with argon2id password hashing algorithm. Lastly, I will demonstrate how to create machine owner key (MOK) to sign grub bootloader and vmlinuz.


## Build Environment
In order to follow along with this guide you will need Debian. Although, you can use this guide (with a few tweaks) with other linux distributions. I will be using debian-live-12.5.0-amd64-xfce.iso to make the commands in this guide flow more smoothly if you build from a fresh Debian live Iso. However, with that said, you should have a basic understanding of linux commands and file structure to troubleshoot. If you are new to Linux I will suggest a free Introduction to Linux course (https://training.linuxfoundation.org/training/introduction-to-linux/) from the Linux Foundation.

'''
user@debian:~$ sudo su -

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

I will create a directory and variable for the mount point. you can use commands such as 'lsusb' or 'dmesg' to identify the USB you plan to build on, but it is much simpler to run 'lsblk' before you plug in the USB, and then again after you plug in the USB. I will be using a 32GB USB, which on my system is '/dev/sdb'

####ADD PROXY TO APT########

cat > /etc/apt/apt.conf.d/95Proxies << EOF
Acquire::http::Proxy "http://192.168.49.1:8000";
Acquire::https::Proxy "http://192.168.49.1:8000";
Acquire::ftp::Proxy "http://192.168.49.1:8000";
EOF

###############################

mkdir -vp /mnt/kaisen

echo 'CB="/mnt/kaisen"' >> ~/.bashrc

source ~/.bashrc

echo $CB
'''




### What you need:
  Debian live ISO (12.5.0 or later)
     - https://www.debian.org/CD/live/
  
  2 x USB 3.0 thumb drives with at least 32GB

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

## Partitioning the USB thumb drive

## Creating the Luks2 encrypted partition

## Creating Physical volume, Volume group, Logical Volume

## Creating the File systems

## Mounting the root and EFI partitions

## Prepare debootstrap to build Kaisen

## Debootstrap Kaisen

## Prepare for chroot

## chroot into Kaisen

  ### Build the system
  
  ### Install base packages
  
  ### Install build dependencies for Grub
  
  ### Create key for initramfs unlock
  
  ### Link key to unlock partition during initramfs stage
  
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
                
