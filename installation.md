# Installing the Arch Linux OS

## Resources

[Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
[Arch Wiki](https://wiki.archlinux.org/)
[Arch Download](https://archlinux.org/download/)
[Reflector](https://wiki.archlinux.org/title/Reflector)
[Encrypting an Entire System](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system)
[Limine](https://wiki.archlinux.org/title/Limine)

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
    - [Encryption](#encryption)
    - [Btrfs and Base System](#btrfs-and-base-system)
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
(this erases the whole stick).

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
| `/dev/efi_system_partition` | `default`    | `4G`        | `ef00` | `boot` |
| `/dev/root_partition`       | `default`    | `remaining` | `8309` | `arch` |

The EFI partition holds the kernels for the bootable snapshots, hence 4G. No
swap partition: aarbs sets up swap in compressed RAM (zram).

After creating the partitions: `write` and `quit`. Format the EFI partition:

```sh
mkfs.fat -F 32 /dev/efi_system_partition
```

### Encryption

Encrypt the root partition (LUKS2) with a strong passphrase. It stays as a
fallback; the YubiKey is added after the installation.

```sh
cryptsetup luksFormat /dev/root_partition
# Open it as /dev/mapper/root. The flags enable TRIM and faster SSD access,
# and --persistent stores them in the header for every boot.
cryptsetup open --allow-discards --perf-no_read_workqueue --perf-no_write_workqueue --persistent /dev/root_partition root
```

### Btrfs and Base System

| Subvolume    | Mount                     | Why separate                       |
| ------------ | ------------------------- | ---------------------------------- |
| `@`          | `/`                       | the system, snapshotted by snapper |
| `@home`      | `/home`                   | not rolled back with the system    |
| `@snapshots` | `/.snapshots`             | survives restoring `@`             |
| `@log`       | `/var/log`                | logs survive a rollback            |
| `@pkg`       | `/var/cache/pacman/pkg`   | package cache, not snapshotted     |
| `@docker`    | `/var/lib/docker`         | containers, not snapshotted        |
| `@libvirt`   | `/var/lib/libvirt/images` | VM images, not snapshotted         |
| `@nix`       | `/nix`                    | Nix store, not snapshotted         |

```sh
mkfs.btrfs -L arch /dev/mapper/root
```

[install.sh](https://github.com/askeko/aarbs/blob/main/install.sh) from aarbs
does the rest. Use `lazarus` (desktop) or `halflight` (laptop) as the hostname:
the dotfiles pick lazarus' monitor setup by hostname. Use something else, like
`archtest`, in a VM.

```sh
curl -O https://raw.githubusercontent.com/askeko/aarbs/main/install.sh
sh install.sh /dev/root_partition /dev/efi_system_partition some_name
```

It automates the rest of the install process, refer to install.sh in aarbs. 
It can be run again after a failure.

### Reboot

```sh
umount -R /mnt
reboot
```

Enter the disk passphrase at the boot splash, then continue with the
[post-installation](post-installation.md).
