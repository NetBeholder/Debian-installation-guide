# Installation Guide for Debian 12 (Stable) | FDE | SSD | LUKS | BTRFS
Debian 12 GNU/Linux Installation Guide (SSD/NVMe, FDE with LUKS, BTRFS and debootstrap)
# Introduction
Essentially, this guide is the collective work of many people, based on whose “cave paintings” I compiled and relatively debugged this guide. So watch the ref and read.


Well, «Debian... Debian never changes».

Go!
## Current status
- _*always*_ beta

## Features
* Stable with kernel from backports
* UEFI
* SSD or NVME disk
* FDE: LUKS1 or LUKS2 with some limitations of current GRUB version.
* BTRFS
* Swapfile instead of swap partition
* GRUB Bootloader

## Notes
Some steps require more investigation, testing and optimization:
- SSD optimization (trim/discard configuration)
- fstab mount options
- SecureBoot steps?
- cryptsetup iter-time value (to find a good balance between security and system startup time)
- ?

Can be easily adapted to install as testing (or sid)

## Disk layout:
* ESP / efi partition: 350M - fat32
* Root partition: 100%FREESPACE - BTRFS <-- LUKSv1 or LUKSv2 container
* BTRFS layout: 

	Subvolume  | Mount point
	-----------|:--------------- 
	@    	   |   / 
	@home      |   /home 
	@snapshots |   /.snapshots 
	@swap      |   /swap
	@log       |   /var/log 
	@apt       |   /var/cache/apt	

# Table of contents
0. [Introduction](#introduction)
    1. [Current status](#current-status)
    2. [Features](#features)
    3. [Notes](#notes)
    4. [Disk layout](#disk-layout)
1. [Initial settings](#initial-settings)
    1. [Live OS](#live-os)
    2. [Logging in (locally)](#logging-in-locally)
    3. [Configuring OpenSSH Server](#configuring-openssh-server)
    4. [Establishing SSH-connection](#establishing-ssh-connection)
    5. [BIOS or UEFI?](#bios-or-uefi)
2. []

    2. [Creating file systems and swap](#creating-file-systems-and-swap)
    	1. [Set environment variables](#set-environment-variables)
        2. [Partitioninig disks](#partitioninig-disks)
        3. [Partitions encryption](#partitions-encryption)
           1. [LUKS-encrypted boot](#luks-encrypted-boot)
           2. [LUKS-encrypted root](#luks-encrypted-root)
           3. [Open encrypted partitions](#open-encrypted-partitions)
	4. [Configuring LVM](#configuring-lvm)
	5. [Formatting partitions](#formatting-partitions)
	6. [Mounting partitions](#mounting-partitions)
	7. [Making swapfile](#making-swapfile)


# Initial settings
## Live OS
The installation method (debootstrap) used in this guide requires internet access. The least problematic way to get online is to use a wired connection.

### Logging in (locally)
The user "user" has the password "live"

### Configuring OpenSSH Server
First, let's set up a SSH server for remote connections and set a password for root.

```bash
sudo -s
# set password for root
passwd
apt install openssh-server
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
systemctl restart sshd
ss -tuln | grep 22
ip a s
```
### Establishing SSH-connection
Initiate SSH-session to the live OS instance with root/"YourPass" credentials from your prefered SSH-client.
```bash
ssh root@IP
#sudo -s
```
### BIOS or UEFI?
To determine the platform, run:
```bash
mount | grep efivars
  efivarfs on /sys/firmware/efi/efivars type efivarfs (rw,nosuid,nodev,noexec,relatime)
  # it is UEFI.
```
## Creating file systems and swap
### Set Environment variables
First check the target disk device (sdX, nvmeXnY).

> _SATA device:_
```bash
lsblk

  NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS 
  loop0     7:0    0 849.1M  1 loop
  loop1     7:1    0     4G  1 loop /run/rootfsbase
  sda       8:0    0    20G  0 disk
  sr0      11:0    1   983M  0 rom  /run/initramfs/live
```
> _NVME device:_
```bash
lsblk

  NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
  loop0     7:0    0 849.1M  1 loop
  loop1     7:1    0     4G  1 loop /run/rootfsbase
  sr0      11:0    1   983M  0 rom  /run/initramfs/live
  nvme0n1 259:0    0    20G  0 disk
```
Then set the environment variables:
```bash
# choose SATA or NVME
# for SATA
export DEV="/dev/sda"
# OR for NVME
export DEV="/dev/nvme0n1"

export DM="${DEV##*/}"
export DEVP="${DEV}$( if [[ "$DEV" =~ "nvme" ]]; then echo "p"; fi )"
export DM="${DM}$( if [[ "$DM" =~ "nvme" ]]; then echo "p"; fi )"

#EFI Partition Number
export EPN=1
#Root Partition Number
export RPN=2
```
### Partitioninig disks
Let's create partitions with cfdisk:
```bash
cfdisk "${DEV}"
  select label type: gpt
  Free space
  [NEW]
  Partition size: 350M  #ESP/efi partition size can be safely set to 100M+. I usually prefer 200-350M.
  Type: EFI System
  Free space
  New
  Partition size: 18.7G (100% free space)
  [Write]
  yes
  [Quit]
```
And check the final scheme.
> _SATA device:_
```bash
lsblk -l

  NAME  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
  loop0   7:0    0 849.1M  1 loop
  loop1   7:1    0     4G  1 loop /run/rootfsbase
  sda     8:0    0    20G  0 disk
  sda1    8:1    0   350M  0 part
  sda2    8:2    0     1G  0 part
  sda3    8:3    0  18.7G  0 part
  sr0    11:0    1   983M  0 rom  /run/initramfs/live
```
> _NVME device:_
```bash
lsblk -l

  NAME      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
  loop0       7:0    0 849.1M  1 loop
  loop1       7:1    0     4G  1 loop /run/rootfsbase
  sr0        11:0    1   983M  0 rom  /run/initramfs/live
  nvme0n1   259:0    0    20G  0 disk
  nvme0n1p1 259:4    0   350M  0 part
  nvme0n1p2 259:5    0     1G  0 part
  nvme0n1p3 259:6    0  18.7G  0 part
```

### Partition encryption
Neither GRUB 2.12 nor GRUB 2.06 currently supports the Argon2id PBKDF.

>Note: Passphrase iteration count is based on time and hence security level depends on CPU power of the system the LUKS container is created on. Depending on security requirements, this may need adjustment.  For LUKS1, you can just look at the iteration count on different systems and select one you like.  You can also change the benchmark time with the -i parameter to create a header for a slower system.
See: https://gitlab.com/cryptsetup/cryptsetup/-/wikis/FrequentlyAskedQuestions#2-setup
#### LUKS-encrypted root

> to create a LUKSv1 container:
```bash
cryptsetup -y -v luksFormat --type=luks1 -i 500 "${DEVP}${RPN}" #-i 1000? More value - longer loading!
```

> to create a LUKSv2 container (pbkdf2):
```bash
cryptsetup --pbkdf pbkdf2 --key-size 512 --hash sha512 luksFormat "${DEVP}${RPN}" 
```
#### Open encrypted partitions
Open the LUKS container now and check the results:
>for LUKSv1:
```bash
cryptsetup open ${DEVP}${RPN} ${DM}${RPN}_crypt --allow-discards
  Enter passphrase for /dev/sda2:
  #Enter passphrase for /dev/nvme0n1p2:
```
>for LUKSv2:
#With LUKS2 you can set --perf-no_read_workqueue and --perf-no_write_workqueue as default flags for a device by opening it once with the option --persistent.
```bash
cryptsetup open ${DEVP}${RPN} ${DM}${RPN}_crypt --allow-discards --perf-no_read_workqueue --perf-no_write_workqueue --persistent 
  Enter passphrase for /dev/sda2:
  #Enter passphrase for /dev/nvme0n1p2:
```
#### Formatting partitions
```bash
mkfs.vfat -nESP -F32 ${DEVP}${EPN}
mkfs.btrfs -L root /dev/mapper/${DM}${RPN}_crypt
```
#### Creating BTRFS subvolumes
```bash
mount_opt="rw,noatime,compress=lzo,ssd,discard=async,space_cache,space_cache=v2,commit=120"
mount -o $mount_opt /dev/mapper/${DM}${RPN}_crypt /mnt
mkdir -p /mnt/boot/efi
mount ${DEVP}${EPN} /mnt/boot/efi/

# Typical BTRFS layout for Debian
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@swap
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@apt
```
Some applications may require their own subvolumes:
```console
# libvirt
btrfs subvolume create /mnt/@libvirt
# podman
#btrfs subvolume create /mnt/@podman
# docker
btrfs subvolume create /mnt/@docker
```
#### Mounting partitions and subvolumes
Create directories and mount the BTRFS subvolumes:
```bash
mkdir -p /mnt/{swap,.snapshots,home,var/log,var/cache/apt}
mount -o subvol=@snapshots,$mount_opt /dev/mapper/${DM}${RPN}_crypt /mnt/.snapshots
mount -o subvol=@home,$mount_opt /dev/mapper/${DM}${RPN}_crypt /mnt/home
mount -o subvol=@swap,$mount_opt /dev/mapper/${DM}${RPN}_crypt /mnt/swap
mount -o subvol=@log,$mount_opt /dev/mapper/${DM}${RPN}_crypt /mnt/var/log
mount -o subvol=@apt,$mount_opt /dev/mapper/${DM}${RPN}_crypt /mnt/var/cache/apt
```
and if needed for some apps:
```bash
mkdir -p /mnt/{var/lib/libvirt/images,var/lib/docker/btrfs}
mount -o subvol=@libvirt,$mount_opt /dev/mapper/${DM}${RPN}_crypt /mnt/var/lib/libvirt/images
mount -o subvol=@docker,$mount_opt /dev/mapper/${DM}${RPN}_crypt /mnt/var/lib/docker/btrfs
#mount -o subvol=@libvirt_images,$mount_opt /dev/mapper/${DM}${RPN}_crypt /mnt/var/lib/libvirt/images
chattr +C -R /mnt/var/lib/libvirt/images/
#
#mount -o subvol=@lib_containers,$mount_opt /dev/mapper/${DM}${RPN}_crypt /mnt/var/lib/containers
#chattr +C -R /mnt/var/lib/containers
#
#mount -o subvol=@docker,$mount_opt /dev/mapper/${DM}${RPN}_crypt /mnt/var/lib/docker/btrfs
#chattr +C -R /mnt/var/lib/docker/btrfs
```

#### Making swapfile
Determining the size of the swap file (partition) can be tricky. If you use hibernation, the swap space size should equal the amount of RAM plus the square root of the RAM amount.
> SwapFileSize = RAM + SQRT(RAM).
Thus, if the RAM size is 4, then the swap partition size should be 6.
```bash
# disabling the copy-on-write feature should also disable compression
chattr +C -R /mnt/swap
#btrfs property set /mnt/swap compression none
btrfs filesystem mkswapfile --size 6G --uuid clear /mnt/swap/swapfile
#mkswap -L SWAP /mnt/swap/swapfile
swapon /mnt/swap/swapfile
```
# System Installation
We are ready to deploy Debian.
## Base system (debootstrap)
```bash
apt install debootstrap arch-install-scripts
debootstrap --arch amd64 stable /mnt
```
## Chroot
```bash
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt /bin/bash
```
## Basic configuration
### Root password ans shell
```bash
passwd root
chsh -s /bin/bash
```
### Hostname and /etc/hosts
```bash
echo "debian-host" > /etc/hostname
#Optionally
cat > /etc/hosts << EOF
127.0.0.1 localhost
127.0.1.1 $(cat /etc/hostname)
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
EOF
```
### Date and time
```bash
export TimeZone="Europe/Moscow"
ln -fs /usr/share/zoneinfo/${TimeZone} /etc/localtime
#ln -fs /usr/share/zoneinfo/Europe/Moscow /etc/localtime
dpkg-reconfigure --frontend noninteractive tzdata
```
### System locales
```bash
apt install locales
#dpkg-reconfigure locales
sed -i 's/# en_US.UTF-8/en_US.UTF-8/g' /etc/locale.gen
sed -i 's/# ru_RU.UTF-8/ru_RU.UTF-8/g' /etc/locale.gen
#locale-gen
echo 'LANG="en_US.UTF-8"' > /etc/default/locale
dpkg-reconfigure --frontend=noninteractive locales
update-locale LANG=en_US.UTF-8
#dpkg-reconfigure locales
```
### Apt sources list
See https://wiki.debian.org/SourcesList.
```bash
apt install lsb-release
mv /etc/apt/sources.list /etc/apt/sources.list.old
CODENAME=$(lsb_release --codename --short)
#stable, non-free, backports
cat > /etc/apt/sources.list << EOF
deb http://deb.debian.org/debian ${CODENAME} main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian ${CODENAME} main contrib non-free non-free-firmware

deb http://deb.debian.org/debian-security/ ${CODENAME}-security main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian-security/ ${CODENAME}-security main contrib non-free non-free-firmware

deb http://deb.debian.org/debian ${CODENAME}-updates main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian ${CODENAME}-updates main contrib non-free non-free-firmware

deb http://deb.debian.org/debian ${CODENAME}-backports main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian ${CODENAME}-backports main contrib non-free non-free-firmware
EOF

apt update
```
### System packages and kernel
For better hardware support (Wi-Fi), I use the kernel from the backports repository. Alternatively, it can be installed from the main repo.
```bash 
apt install btrfs-progs dosfstools cryptsetup-initramfs grub-efi cryptsetup-suspend firmware-linux firmware-linux-nonfree sudo neovim bash-completion command-not-found plocate systemd-timesyncd fonts-terminus# usbutils hwinfo

#install kernel from backports
#apt install -t ${CODENAME}-backports linux-image-amd64 linux-headers-amd64
apt install linux-image-amd64/${CODENAME}-backports \
 linux-headers-amd64/${CODENAME}-backports

#OR from main repo
#apt install linux-image-amd64 linux-headers-amd64
```
### Encryption settings
#### Environment variables
Setting the ​​environment variables:
```bash
export LUKS_UUID=$(blkid -s UUID -o value /dev/"${DM}${RPN}")
export SWAPFILE_UUID=`findmnt -no UUID -T /swap/swapfile`
export SWAPFILE_OFFSET=`btrfs inspect-internal map-swapfile -r /swap/swapfile`
export cryptkey=/etc/keys/root.key
```
#### LUKS Keyfile
```bash
mkdir -m0700 /etc/keys
umask 0077 && dd if=/dev/urandom bs=1 count=64 of=${cryptkey} conv=excl,fsync
cryptsetup luksAddKey ${DEVP}${RPN} $cryptkey
#cryptsetup luksDump ${DEVP}${RPN} | grep "^Key Slot"

#dd bs=512 count=4 if=/dev/urandom of=$cryptkey
#cryptsetup luksAddKey ${DEVP}${RPN} $cryptkey
#chmod 600 $cryptkey
```
#### crypttab
#crypttab. Edit the crypttab(5) and set the third column to the key file path for the root device entry.
```bash
echo "${DM}${RPN}_crypt UUID=$(blkid -s UUID -o value ${DEVP}${RPN}) $cryptkey luks,discard,key-slot=1" >> /etc/crypttab
```
#### initramfs
```bash
#In /etc/cryptsetup-initramfs/conf-hook, set KEYFILE_PATTERN to a glob(7) expanding to the key path names to include to the initramfs image.
echo "KEYFILE_PATTERN=\"/etc/keys/*.key\"" >>/etc/cryptsetup-initramfs/conf-hook
#In /etc/initramfs-tools/initramfs.conf, set UMASK to a restrictive value to avoid leaking key material. See initramfs.conf(5) for details.
echo UMASK=0077 >> /etc/initramfs-tools/initramfs.conf
#Finally re-generate the initramfs image, and double-check that it 1/ has restrictive permissions; and 2/ includes the key.
update-initramfs -u #update-initramfs -c -k all
```
### Bootloader
```bash
#grub config
sed -i "/^GRUB_CMDLINE_LINUX_DEFAULT=/ s/\(\"[^\"]*\)$/ cryptdevice=UUID=${LUKS_UUID}:${DM}${RPN}_crypt root=\/dev\/mapper\/${DM}${RPN}_crypt resume=UUID=$SWAPFILE_UUID resume_offset=$SWAPFILE_OFFSET&/" /etc/default/grub

echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub
echo 'GRUB_PRELOAD_MODULES="part_gpt part_msdos cryptodisk luks btrfs"' >> /etc/default/grub #btrfs?
#optional graphics-related grub settings
echo 'GRUB_GFXPAYLOAD=keep' >> /etc/default/grub
echo 'GRUB_TERMINAL=gfxterm' >> /etc/default/grub
echo 'GRUB_GFXMODE=1920x1080x32' >> /etc/default/grub

update-grub
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Debian --recheck
```

### Finalizing OS Installation
Leave chroot environment and reboot the machine.
```bash
exit
umount -a
reboot
```
