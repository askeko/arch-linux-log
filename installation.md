# Installing the Arch Linux OS

## Resources

[Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
[Arch Wiki](https://wiki.archlinux.org/)
[Arch Download](https://archlinux.org/download/)
[Reflector](https://wiki.archlinux.org/title/Reflector)

<!--toc:start-->
- [Installing the Arch Linux OS](#installing-the-arch-linux-os)
  - [Resources](#resources)
  - [Pre-installation](#pre-installation)
    - [Installation Image](#installation-image)
    - [Verify Checksum](#verify-checksum)
    - [Prepare Installation Medium](#prepare-installation-medium)
  - [Installation](#installation)
    - [Keyboard Layout](#keyboard-layout)
    - [Font](#font)
    - [Check UEFI](#check-uefi)
    - [Network Setup](#network-setup)
      - [Optional connect via Wi-Fi](#optional-connect-via-wi-fi)
    - [Set System Clock](#set-system-clock)
    - [Partitioning](#partitioning)
    - [Configure Mirrors](#configure-mirrors)
    - [Installing the Base System](#installing-the-base-system)
    - [Time](#time)
    - [Localization](#localization)
    - [Internet connection](#internet-connection)
    - [Hostname](#hostname)
    - [Bootloader](#bootloader)
    - [Password](#password)
    - [Reboot](#reboot)
<!--toc:end-->

## Pre-installation

### Installation Image

Download the latest Arch Linux release and its signature.

### Verify Checksum

```sh
# Replace version with the version of the installation image
sha256sum archlinux-version-x86_64.iso # compare with the download page
gpg --keyserver-options auto-key-retrieve --verify archlinux-version-x86_64.iso.sig
```

### Prepare Installation Medium

Write the ISO to a USB stick, replacing the path and drive appropriately
(this erases the whole stick). Boot via UEFI with Secure Boot disabled.

```sh
sudo dd bs=4M if=path/to/archlinux.iso of=/dev/sdx status=progress oflag=sync
```

## Installation

### Keyboard Layout

```sh
loadkeys dk
```

### Font

```sh
setfont ter-v22n
```

### Check UEFI

Verify UEFI mode enabled, it should return 64:

```sh
cat /sys/firmware/efi/fw_platform_size
```

### Network Setup

Verify network is connected (wired):

```sh
ping archlinux.org
```

#### Optional connect via Wi-Fi

```sh
iwctl # for the wi-fi interactive prompt
[iwctl] device list # list devices
[iwctl] station <device> scan # scan for networks
[iwctl] station <device> get-networks # list scanned networks
[iwctl] station <device> connect <SSID> # connect to network
```

### Set System Clock

```sh
timedatectl set-ntp true
timedatectl status # check that the clock is synchronized
```

### Partitioning

Use `lsblk -f` to identify your drives. Don't touch lazarus' data SSD.

Erase all data on a drive (BE CAREFUL!!!):

```sh
sgdisk --zap-all /dev/X # Replace 'X' with your drive. (Usually nvmeX or sdaX)
```

Setup partitions using `cgdisk`:

```sh
cgdisk /dev/X
```

| Partition                   | First sector | Size        | Type   | Name   |
| --------------------------- | ------------ | ----------- | ------ | ------ |
| `/dev/efi_system_partition` | `default`    | `1G`        | `ef00` | `boot` |
| `/dev/swap_partition`       | `default`    | `8G`        | `8200` | `swap` |
| `/dev/root_partition`       | `default`    | `remaining` | `8300` | `arch` |

After creating the partitions `write` and `quit`. Then format and mount them:

```sh
# Format partitions:
mkfs.ext4 /dev/root_partition
mkswap /dev/swap_partition
mkfs.fat -F 32 /dev/efi_system_partition
# Mount partitions
mount /dev/root_partition /mnt
mount --mkdir /dev/efi_system_partition /mnt/boot
# Enable swap
swapon /dev/swap_partition
```

Verify with `lsblk`.

### Configure Mirrors

Rank nearby mirrors before `pacstrap`, which copies the mirrorlist into the new
system. The ISO's default ranking can put a slow mirror from another continent
first. See [Reflector].

```sh
reflector --country Denmark,Germany,Sweden --protocol https --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
```

### Installing the Base System

Install essential packages (replace amd for intel if necessary, skip the
microcode in a VM). aarbs installs everything else later.

```sh
pacstrap -K /mnt base linux linux-firmware amd-ucode networkmanager neovim grub efibootmgr curl
```

Generate fstab and enter chroot:

```sh
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab # Optionally check if fstab was generated correctly
arch-chroot /mnt
```

### Time

```sh
ln -sf /usr/share/zoneinfo/Europe/Copenhagen /etc/localtime
hwclock --systohc
systemctl enable systemd-timesyncd.service
```

### Localization

Uncomment `en_DK.UTF-8 UTF-8` and `en_US.UTF-8 UTF-8`:

```sh
nvim /etc/locale.gen
locale-gen # generate uncommented locales
echo LANG=en_DK.UTF-8 > /etc/locale.conf
echo KEYMAP=dk > /etc/vconsole.conf
```

### Internet connection

```sh
systemctl enable NetworkManager.service
```

### Hostname

Use `lazarus` (desktop) or `halflight` (laptop): the dotfiles pick lazarus'
monitor setup by hostname. Use something else, like `archtest`, in a VM.

```sh
echo some_name > /etc/hostname # replace some_name with the hostname
```

```sh
/etc/hosts
-------------
127.0.0.1 localhost
::1 localhost
127.0.1.1 some_name.localdomain some_name # replace some_name with the hostname
```

### Bootloader

```sh
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg # Make the grub config file
```

### Password

Set root password:

```sh
passwd
```

### Reboot

Exit chroot and reboot:

```sh
exit
umount -R /mnt
reboot
```

Continue with the [post-installation](post-installation.md).
