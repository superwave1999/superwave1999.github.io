---
title: "Ubuntu Server 24.04.2 with ZFS root"
date: 2025-06-19
tags: [ubuntu, zfs, zfs-root, zfsbootmenu, refind, server, linux, installation, guide, bootloader, networking, homelab]
categories: [documentation]
---

# Introduction & Prerequisites

Hello! This guide will guide you through an installation of Ubuntu Server 24.04.2 LTS.

> I will update this guide if I find any further caveats. The main issue I can imagine popping up is Ubuntu assuming we are running the grub2 bootloader instead of rEFInd and zfsbootmenu.

The target machine specs and the reasoning behind doing this (and for choosing ZFS over BTRFS) are detailed [here](/posts/setup-tour).

Before following this guide, I suggest you read through [the original](https://docs.zfsbootmenu.org/en/v2.3.x/guides/ubuntu/noble-uefi.html). The only difference is that in this case, we will be installing this on a server machine with no desktop and on a blank SSD.

I will copy-paste most of the original commands, but with some changes (prefixed with ⭐) and comments (prefixed with 💬).

This process will take around 1 hour for those relatively technically-minded.

You should have an Ubuntu USB flashed using balena, rufus, or any other trustworthy tool plugged into the back of the motherboard.

You should also be able to boot into this USB drive.

Right now you should see the initial language selection screen of the installer.

## Basic Rundown

- We boot the installer USB
- We enter the command-line (bash)
- We format our drive and prepare / mount ZFS
- With ZFS mounted to the live environment, we install Ubuntu manually to the ZFS mount.
- We install a ZFS-capable bootloader
- We install some basic packages that most of us will use in a server environment.
- Reboot

## Caveats

The original guide assumes you know the inner workings of the ubuntu installer and what it does in the background.

- A relatively-open root user will be created automatically. We will lock it down later.
- No nano, SSH, nor internet unless configured.
- I'm running ethernet, steps may be different for Wi-Fi.

### ⭐ Enter the command-line

From the ubuntu server installer, you can navigate using the keyboard to the "Help" dropdown at the top right of the screen. From here you can use the command-line sitting at the machine, or SSH into it. I did the latter for convenience to copypaste the commands of the guide.

### Open a root shell
💬 Skipping this original section, you should be as root user via SSH.

### Prepare the current command-line session
Load ubuntu release info and codenames into the session. We will also install some additional utilities into the live environment.

```bash
source /etc/os-release
export ID
apt update
apt install debootstrap gdisk zfsutils-linux
zgenhostid -f 0x00bab10c
```

### ⭐ 💬 Detect your disk partitions
💬 Arguably the most important part of the guide. You must know which type of boot drive you have, but since some M.2 drives use a SATA interface and not PCI-E, we will check.

```bash
lsblk
```
In my case with the installer on USB and with a PCI-E M.2 SSD, the output looks like this:

```bash
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0    7:0    0   2.3G  1 loop /rofs
sda      8:0    1  14.5G  0 disk
├─sda1   8:1    1   2.7G  0 part /cdrom
├─sda2   8:2    1   4.0M  0 part
nvme0n1 259:0    0 512.1G  0 disk
├─nvme0n1p1 259:1  512M  0 part
├─nvme0n1p2 259:2 511.6G  0 part

```
For a traditional SATA SSD, HDD, or SATA M.2, it will look like the following:
```bash
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0    7:0    0   2.3G  1 loop /rofs
sda      8:0    1  14.5G  0 disk
├─sda1   8:1    1   2.7G  0 part /cdrom
├─sda2   8:2    1   4.0M  0 part
sdb      8:16   0 512.1G  0 disk
├─sdb1   8:17   0 500.0G  0 part
├─sdb2   8:18   0  12.1G  0 part

```

The main things you should look out for are the /cdrom mountpoint (this is the live USB). The other non-loop drive (assuming you only have 1 permanent drive in your system) should be the target drive. To double-confirm, you can also compare the sizes of the drives for a relative match.

### Wipe and Configure your disk partitions
💬 BOOT is where the rEFInd and zfsbootmenu go. The POOL is the OS.

```bash
export BOOT_DISK="/dev/nvme0n1"
export BOOT_PART="1"
export BOOT_DEVICE="${BOOT_DISK}p${BOOT_PART}"
export POOL_DISK="/dev/nvme0n1"
export POOL_PART="2"
export POOL_DEVICE="${POOL_DISK}p${POOL_PART}"

```

Wipe and partition the drives:
```bash
zpool labelclear -f "$POOL_DISK"
wipefs -a "$POOL_DISK"
wipefs -a "$BOOT_DISK"
sgdisk --zap-all "$POOL_DISK"
sgdisk --zap-all "$BOOT_DISK"
sgdisk -n "${BOOT_PART}:1m:+512m" -t "${BOOT_PART}:ef00" "$BOOT_DISK"
sgdisk -n "${POOL_PART}:0:-10m" -t "${POOL_PART}:bf00" "$POOL_DISK"
```

### ZFS pool creation + filesystems
💬 I disabled compression in my case.

```bash
zpool create -f -o ashift=12 \
 -O compression=off \
 -O acltype=posixacl \
 -O xattr=sa \
 -O relatime=on \
 -o autotrim=on \
 -o compatibility=openzfs-2.1-linux \
 -m none zroot "$POOL_DEVICE"

zfs create -o mountpoint=none zroot/ROOT
zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/${ID}
zfs create -o mountpoint=/home zroot/home
zpool set bootfs=zroot/ROOT/${ID} zroot
```

### Export, then re-import with a temporary mountpoint of `/mnt` & update device symlinks
```bash
zpool export zroot
zpool import -N -R /mnt zroot
zfs mount zroot/ROOT/${ID}
zfs mount zroot/home
udevadm trigger
```

### Install Ubuntu
💬 This is the longest part of this process, it may appear stuck since there is no progress indicator. To clarify, we are installing Ubuntu to our ZFS'ed system drive mounted to /mnt in the previous step.

```bash
debootstrap noble /mnt
```

### Copy files
```bash
cp /etc/hostid /mnt/etc
cp /etc/resolv.conf /mnt/etc
```

### Chroot into the new OS
💬 What is chroot you ask? Simple! So far, any changes that we make to the system were changes made to the live environment, that lives in RAM, and disappears on reboot. With the following steps, we are applying changes to the recently installed system on the SSD. All without rebooting. Ah... the wonders of Linux.

```bash
mount -t proc proc /mnt/proc
mount -t sysfs sys /mnt/sys
mount -B /dev /mnt/dev
mount -t devpts pts /mnt/dev/pts
chroot /mnt /bin/bash
```

### Postinstall steps
#### Set hostname
💬 Replace YOURHOSTNAME with something like server, homelab... whatever you want.
```bash
echo 'YOURHOSTNAME' > /etc/hostname
echo -e '127.0.1.1\tYOURHOSTNAME' >> /etc/hosts
```

#### Set a root password
```bash
passwd
```

#### Configure apt
```bash
cat <<EOF > /etc/apt/sources.list
# Uncomment the deb-src entries if you need source packages

deb http://archive.ubuntu.com/ubuntu/ noble main restricted universe multiverse
# deb-src http://archive.ubuntu.com/ubuntu/ noble main restricted universe multiverse

deb http://archive.ubuntu.com/ubuntu/ noble-updates main restricted universe multiverse
# deb-src http://archive.ubuntu.com/ubuntu/ noble-updates main restricted universe multiverse

deb http://archive.ubuntu.com/ubuntu/ noble-security main restricted universe multiverse
# deb-src http://archive.ubuntu.com/ubuntu/ noble-security main restricted universe multiverse

deb http://archive.ubuntu.com/ubuntu/ noble-backports main restricted universe multiverse
# deb-src http://archive.ubuntu.com/ubuntu/ noble-backports main restricted universe multiverse
EOF

apt update
apt upgrade
```

#### Additional base packages
💬 Note the --no-install-recommends. These commands configure command-line languages and keyboards. As the original guide says: always enable the en_US.UTF-8 locale because some programs require it.

```bash
apt install --no-install-recommends linux-generic locales keyboard-configuration console-setup
dpkg-reconfigure locales tzdata keyboard-configuration console-setup
```

💬 Also, some packages we installed on the live USB must also be installed to the full install.
```bash
apt install dosfstools zfs-initramfs zfsutils-linux zstd
systemctl enable zfs.target
systemctl enable zfs-import-cache
systemctl enable zfs-mount
systemctl enable zfs-import.target
update-initramfs -c -k all
```

#### ⭐ 💬 Additional package recommendations & current state of the system

Note! Here be dragons. YMMV.

I want a very lean setup in my case, so I installed the ubuntu-server-minimal package, and maybe it's too aggressive. When I rebooted (don't do it yet) I didn't have internet configured, nor an SSH server to remote in. So, I'll install standard ubuntu-server package.

I'll post some troubleshooting steps I had to take at the end of this guide.

```bash
apt install --no-install-recommends ubuntu-server nano btop openssh-server ncurses-term ssh-import-id
```

### Install and configure the bootloader
```bash
zfs set org.zfsbootmenu:commandline="quiet" zroot/ROOT
mkfs.vfat -F32 "$BOOT_DEVICE"

cat << EOF >> /etc/fstab
$( blkid | grep "$BOOT_DEVICE" | cut -d ' ' -f 2 ) /boot/efi vfat defaults 0 0
EOF

mkdir -p /boot/efi
mount /boot/efi

apt install curl

mkdir -p /boot/efi/EFI/ZBM
curl -o /boot/efi/EFI/ZBM/VMLINUZ.EFI -L https://get.zfsbootmenu.org/efi
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/ZBM/VMLINUZ-BACKUP.EFI

mount -t efivarfs efivarfs /sys/firmware/efi/efivars

## Below press "Yes"
apt install refind
refind-install
rm /boot/refind_linux.conf

cat << EOF > /boot/efi/EFI/ZBM/refind_linux.conf
"Boot default"  "quiet loglevel=0 zbm.skip"
"Boot to menu"  "quiet loglevel=0 zbm.show"
EOF
```

### First boot
```bash
umount -n -R /mnt

zpool export zroot
reboot
```

# ⭐ 💬 Post-first-boot and potential troubleshooting steps
### Create a non-root user and disable the root account
It's wise from a security standpoint to disable the root user. From the root account, run the following commands:

```bash
adduser --uid 1000 yourusername
usermod -aG sudo yourusername
passwd -l root

nano /etc/ssh/sshd_config.d/10-disable-root.conf
## Write in the file and save:
PermitRootLogin no

systemctl reload sshd
reboot
```
After rebooting, you can only log in via your non-root user, and can then elevate privileges.


### Help! I have no internet after rebooting!
One of the steps the ubuntu installer does is set up the internet for you, but we didn't use the ubuntu installer.

You will need to sit at the server for a bit until we get internet working.
Note, IP addresses or interface names may change in your case. This configuration was done via ethernet.

#### 1. Assign a known IP address (temporary)
```bash
ip addr add 192.168.1.50/24 dev enp2s0
ip route add default via 192.168.1.1
```
#### 2. Make changes permanent by configuring netplan
```bash
nano /etc/netplan/01-netcfg.yaml
```
Paste the following, changing your adapter name:
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    all:
      match:
        name: "en*"
      dhcp4: true
      dhcp6: true
```

Save, set permissions, apply changes:
```bash
chmod 600 /etc/netplan/01-netcfg.yaml
netplan generate
netplan apply

systemctl enable systemd-networkd
systemctl enable systemd-resolved
systemctl restart systemd-networkd
systemctl restart systemd-resolved
```

Reboot to see if it's automatically configured.

# Conclusion

Thats it! You should have a working system prepared for ZFS snapshots and with internet access ;)

In following guides, I'll debloat the system and set up backups of the snapshots, simulating disaster scenarios.