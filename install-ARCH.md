# Install ARCH Linux - MY WAY 
Features:
- encrypted (with sd-encrypt)
- systemd-boot

The official installation guide (https://wiki.archlinux.org/index.php/Installation_Guide) contains a more verbose description.

## Prerequisites
- Download the archiso image from https://www.archlinux.org/
- Copy to a usb-drive
```
dd if=archlinux.img of=/dev/sdX bs=16M && sync # on linux
```
- Boot from the usb. If the usb fails to boot, make sure that secure boot is disabled in the BIOS configuration.

## Set keymap
```
loadkeys de-latin1
```

## Configure wifi 
guide: https://wiki.archlinux.org/index.php/Iwd#iwctl
```
iwctl
```

## Create partitions
```
fdisk /dev/sdX
```
- g # create GPT partition table
- n # new EFI partition 
- +512M # size
- t # change type
- 1 # EFI partition type (code EF)
- n # new partition for root
- p # print
- w # write

```
mkfs.vfat -F32 /dev/sdX1
mkfs.ext2 /dev/sdX2
```

## Setup the encryption of the system
```
cryptsetup luksFormat /dev/sdX2
cryptsetup luksOpen /dev/sdX2 luks
```

## Create encrypted partitions
This creates one partions for root, modify if /home or other partitions should be on separate partitions
```
pvcreate /dev/mapper/luks
vgcreate vg0 /dev/mapper/luks
# lvcreate --size 8G vg0 --name swap
lvcreate -l +100%FREE vg0 --name root
```
### Create filesystems on encrypted partitions
```
mkfs.ext4 /dev/mapper/vg0-root
# mkswap /dev/mapper/vg0-swap
```

### Mount the new system 
```
mount /dev/mapper/vg0-root /mnt # /mnt is the installed system
# swapon /dev/mapper/vg0-swap # Not needed but a good thing to test
mkdir /mnt/boot
mount /dev/sdX1 /mnt/boot
```

## Install the system 
also includes some optional stuff
```
pacstrap /mnt base base-devel linux linux-firmware lvm2 vim nano git iwd htop
```

## Generate fstab
```
genfstab -pU /mnt >> /mnt/etc/fstab
```

### Optional: Make /tmp a ramdisk (add the following line to /mnt/etc/fstab)
```
tmpfs	/tmp	tmpfs	defaults,noatime,mode=1777	0	0
```
Change relatime on all non-boot partitions to noatime (reduces wear if using an SSD)

## Enter the new system
```
arch-chroot /mnt
```

## Setup system clock
```
ln -s /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc
```

## Set the hostname
```
echo MYHOSTNAME > /etc/hostname
```

## Update locale
```
echo en_US.UTF-8 >> /etc/locale.gen # or just uncomment the line in the file
locale-gen
echo LANG=en_US.UTF-8 >> /etc/locale.conf
echo KEYMAP=de-latin1 >> /etc/vconsole.conf
```

## Set password for root
```
passwd
```

### Add my user
```
useradd -m -g users -G wheel -s /bin/bash MYUSERNAME
passwd MYUSERNAME
```

## Configure mkinitcpio with modules needed for the initrd image
https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
```
nano /etc/mkinitcpio.conf
```
Add `systemd` `keyboard` `sd-vconsole` `sd-encrypt` `sd-lvm2` before filesystems.

Example: `HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt sd-lvm2 filesystems fsck)`

### Regenerate initrd image
```
mkinitcpio -p linux
```

## setup **systemd-boot** as bootloader (https://wiki.archlinux.org/index.php/Systemd-boot)
```
bootctl install
```

### add file __/boot/loader/loader.conf__
```
default  arch
timeout  3
```

### add file __/boot/loader/entries/arch.conf__
```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options rd.luks.name={UUID}=vg0 root=/dev/mapper/vg0-root
```
In order to get the UUID run the following command in vim: `:read ! blkid /dev/sda2`

## Exit new system and go into the cd shell
```
exit
```

### Unmount all partitions
```
umount -R /mnt
```
### Reboot into the new system, don't forget to remove the cd/usb
```
reboot
```

