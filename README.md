# About
This repo contains the configuration files I use on my laptop running Arch
Linux. It is updated whenever my laptops local configuration files change.

> The following installation instructions are written for my particular laptop,
> although most of it can be used on any computer.

# Installation of base system
> Mostly taken from
> [ArchWiki](http://wiki.archlinux.org/index.php/Installation_guide) and
> [Btrfs Guide](https://andrewwong.id.au/blog/pure-btrfs-installation/)

## Set the keyboard layout and proper font
```
loadkeys hu
setfont lat9w-16
```

## Verify the boot mode
Arch Linux has booted into UEFI mode if the following folder exists:
```
ls /sys/firmware/efi/efivars
```

## Connect to the Internet
Check if a connection already exists (for wired connection).
```
ping google.com
```

If it doesn't, connect via wifi-menu.
```
wifi-menu
```

## Update the system clock
```
timedatectl set-ntp true
```

Check the system time using:
```
timedatectl status
```

## Partition the disk
Identify the device to install to using either of the following:
```
lsblk -f
fdisk -l
```

Use `cfdisk` or some other tool like `fdisk` or `gdisk` to partition the
drive to your needs.

For example:

| ID        | Type             | Size               |
| --------- | ---------------- | ------------------ |
| /dev/sda1 | EFI System       | 512 MB             |
| /dev/sda2 | Linux Swap       | 16 GB              |
| /dev/sda3 | Linux Filesystem | *rest of the disk* |

## Format the partitions
```
mkfs.vfat -n Boot /dev/sda1
mkswap -L Swap /dev/sda2
mkfs.btrfs -L Arch /dev/sda3
```

Setup Btrfs volumes.
```
mount /dev/sda3 /mnt
cd /mnt
btrfs subvolume create __active
btrfs subvolume create __active/rootvol
btrfs subvolume create __active/home
btrfs subvolume create __active/var
btrfs subvolume create __snapshots

cd ~
umount /mnt
mount -o subvol= __active/rootvol /dev/sda3 /mnt
mkdir /mnt/{home,var}
mount -o subvol= __active/home /dev/sda3 /mnt/home
mount -o subvol= __active/var /dev/sda3 /mnt/var
```

Mount the rest of the partitions.
```
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
swapon /dev/sda2
```

## Install the base system packages
Select the mirrors.
```
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.orig
wget -O /etc/pacman.d/mirrorlist https://www.archlinux.org/mirrorlist/all/
nano /etc/pacman.d/mirrorlist
```

Install the base packages.
```
pacstrap /mnt base base-devel
```

## Configure the system

### Fstab
Generate an `fstab` file.
```
genfstab -U /mnt >> /mnt/etc/fstab
```

### Chroot
Change root into the new system.
```
arch-chroot /mnt
```

### Time zone
Set the time zone:
```
timedatectl list-timezones
ln -s /usr/share/zoneinfo/*Zone*/*SubZone* /etc/localtime
hwclock --systohc --utc
```

### Locale
Uncomment the desired locales in `/etc/locale.gen`.

Generate the selected locales.
```
locale-gen
```

Edit the locale settings in `/etc/locale.conf`.

Set the keyboard layout and font in `/etc/vconsole.conf`:

### Hostname
Create `/etc/hostname` with the desired hostname.

Add a matching entry to `/etc/hosts`.

### Wireless
```
pacman -S  iw wpa_suplicant dialog
```

### Root password
Set the root password with `passwd`.

### Btrfs
```
mkdir /mnt/defvol
```

Edit `/etc/fstab` to mount the default subvolume on boot.

```
pacman -S btrfs-progs
```

### Boot loader
Install the following packages:
```
pacman -S grub efibootmgr intel-ucode
```

Install Grub to the ESP.
```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
```

Generate the main configuration file.
```
grub-mkconfig -o /boot/grub/grub.cfg
```

Copy the Grub EFI file to the Windows EFI file location (for pesky UEFIs).
```
mkdir /boot/EFI/Microsoft
mkdir /boot/EFI/Microsoft/Boot
cp /boot/EFI/grub/grubx64.efi /boot/EFI/Microsoft/Boot/bootmgfw.efi
```

## Reboot
Exit the chroot environment using `exit`.

Manually unmount all partitions.
```
umount -R /mnt
```

Restart the machine using `reboot`.
