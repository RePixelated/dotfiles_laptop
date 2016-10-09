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

# Post-Installation

## Connect to the Internet
Create a wireless connection profile.
```
wifi-menu -o
nano /etc/netctl/*Profile*
```

Create a wired connection profile.
```
nano /etc/netctl/_wired
```

Install and enable the netctl services handling automatic network switching.
```
pacman -S ifplugd wpa_actiond
systemctl enable netctl-auto@wlp2s0.service
systemctl enable netctl-ifplugd@enp4s0.service
```

## User management
Install zsh.
```
pacman -S zsh zsh-completions
```

Setup a new user.
```
useradd -m -g users -G wheel -s /bin/zsh zalan
passwd zalan
chfn zalan
```

Install sudo and set it up.
```
pacman -S sudo
EDITOR=nano visudo
```

Login to the new user and skip the zsh setup.
```
exit
```

## Hardware

### Graphics
Install the Nvidia and Intel drivers and bumblebee.
```
sudo pacman -S bbswitch bumblebee mesa xf86-video-intel nvidia
```

Enable the multilib repository and install the following for 32-bit support.
```
nano /etc/pacman.conf
sudo pacman -S lib32-virtualgl lib32-nvidia-utils lib32-mesa-libgl
```

Add your regular user to the `bumblebee` group.
```
sudo gpasswd -a zalan bumblebee
```

Enable `bumblebeed.service`.

Uncomment the `BusID` line in `/etc/bumblebee/xorg.conf.nvidia` using the
decimal form of the hexadecimal Bus ID in `lspci`.

Install Xorg.
```
sudo pacman -S xorg-server xorg-server-utils xorg-xinit
```

Install your [window manager](README.md#awesome-wm) and
[terminal](README.md#termite) and set `.xinitrc` up.

Test if bumblebee is working.
```
sudo pacman -S mesa-demos
optirun glxgears -info
```

Install `primus`.
```
sudo pacman -S primus lib32-primus
primusrun glxgears
```

Set the `BRIDGE` used by *optirun* to `primus` in
`/etc/bumblebee/bumblebee.conf`.

### Battery, Power management
Install `acpi` to read the battery status using the `-b` flag.
```
sudo pacman -S acpi acpi_call
```

### Touchpad
Install the Synaptics input driver.
```
sudo pacman -S xf86-input-synaptics
```

Copy the configuration file and edit it according to preference.
```
sudo cp /usr/share/X11/xorg.conf.d/70-synaptics.conf /etc/X11/xorg.conf.d/70-synaptics.conf
sudo nano /etc/X11/xorg.conf.d/70-synaptics.conf
```

### Keyboard
Change the keyboard layout in X11.
```
sudo localectl --no-convert set-x11-keymap hu,us pc105 grp:alt_caps_toggle
```

### Audio
Install the ALSA Utilities.
```
sudo pacman -S alsa-utils
```

Run `alsamixer` and use `m` to unmute the *Master* audio channel and use the
up/down arrow keys to set the volume.

## User Interface

### Awesome WM
Install Awesome.
```
sudo pacman -S awesome
```

Edit the `.xinitrc` file to start `awesome`.

### Termite
Install Termite.
```
sudo pacman -S termite
```

Edit the configuration file in `~/.config/termite/config`

### Zsh
Create a redirecting `.zshenv`.
```
nano ~/.zshenv
```

Create a folder for the Zsh configuration files.
```
mkdir ~/.zsh
```

Create a redirecting `zshdir`.
```
nano ~/.zsh/zshdir
chmod 777 ~/.zsh/zshdir
```

Create the following symbolic links:
```
cd ~/.zsh

ln -s zshdir zlogin
ln -s zshdir zlogout
ln -s zshdir zprofile
ln -s zshdir zshenv
ln -s zshdir zshrc

cd ~
```

And create their respective directories.
```
mkdir ~/.zsh/zshenv.d
mkdir ~/.zsh/zshrc.d
```
