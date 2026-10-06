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
    - [Btrfs Subvolumes](#btrfs-subvolumes)
    - [Configure Mirrors](#configure-mirrors)
    - [Installing the Base System](#installing-the-base-system)
    - [Time](#time)
    - [Localization](#localization)
    - [Internet connection](#internet-connection)
    - [Hostname](#hostname)
    - [Initramfs](#initramfs)
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
| `/dev/efi_system_partition` | `default`    | `4G`        | `ef00` | `boot` |
| `/dev/root_partition`       | `default`    | `remaining` | `8309` | `arch` |

The EFI partition holds the kernels for the bootable snapshots, hence 4G. No
swap partition: aarbs sets up swap in compressed RAM (zram).

After creating the partitions `write` and `quit`. Format the EFI partition:

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

### Btrfs Subvolumes

| Subvolume    | Mount                     | Why separate                       |
| ------------ | ------------------------- | ---------------------------------- |
| `@`          | `/`                       | the system, snapshotted by snapper |
| `@home`      | `/home`                   | not rolled back with the system    |
| `@snapshots` | `/.snapshots`             | survives restoring `@`             |
| `@log`       | `/var/log`                | logs survive a rollback            |
| `@pkg`       | `/var/cache/pacman/pkg`   | package cache, not snapshotted     |
| `@docker`    | `/var/lib/docker`         | containers, not snapshotted        |
| `@libvirt`   | `/var/lib/libvirt/images` | VM images, not snapshotted         |

```sh
mkfs.btrfs -L arch /dev/mapper/root
mount /dev/mapper/root /mnt
for sv in @ @home @snapshots @log @pkg @docker @libvirt; do btrfs subvolume create /mnt/$sv; done
umount /mnt
```

Mount them, and the EFI partition at `/boot`:

```sh
mount -o noatime,compress=zstd,subvol=@ /dev/mapper/root /mnt
mount --mkdir -o noatime,compress=zstd,subvol=@home /dev/mapper/root /mnt/home
mount --mkdir -o noatime,compress=zstd,subvol=@snapshots /dev/mapper/root /mnt/.snapshots
mount --mkdir -o noatime,compress=zstd,subvol=@log /dev/mapper/root /mnt/var/log
mount --mkdir -o noatime,compress=zstd,subvol=@pkg /dev/mapper/root /mnt/var/cache/pacman/pkg
mount --mkdir -o noatime,compress=zstd,subvol=@docker /dev/mapper/root /mnt/var/lib/docker
mount --mkdir -o noatime,subvol=@libvirt /dev/mapper/root /mnt/var/lib/libvirt/images
chattr +C /mnt/var/lib/libvirt/images # no copy-on-write for VM images
mount --mkdir /dev/efi_system_partition /mnt/boot
```

Verify with `lsblk`.

### Configure Mirrors

Rank nearby mirrors before `pacstrap`, which copies the mirrorlist into the new
system. The ISO's default ranking can put a slow mirror from another continent
first. See [Reflector](https://wiki.archlinux.org/title/Reflector).

```sh
reflector --country Denmark,Germany,Sweden --protocol https --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
```

### Installing the Base System

Install essential packages (replace amd for intel if necessary, skip the
microcode in a VM). `libfido2` (YubiKey unlock) and `plymouth` (boot splash)
go into the initramfs. aarbs installs everything else later.

```sh
pacstrap -K /mnt base linux linux-firmware amd-ucode btrfs-progs libfido2 plymouth limine efibootmgr networkmanager neovim curl
```

Generate fstab and enter chroot:

```sh
genfstab -U /mnt >> /mnt/etc/fstab
sed -i 's/,subvolid=[0-9]*//' /mnt/etc/fstab # mount by name, so a restored @ is used
sed -i '/[[:space:]]\/boot[[:space:]]/s/fmask=0022,dmask=0022/fmask=0077,dmask=0077/' /mnt/etc/fstab # ESP readable by root only (random seed)
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
echo KEYMAP=dk > /etc/vconsole.conf # also the layout for the disk passphrase
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

### Initramfs

The initramfs unlocks the disk (`sd-encrypt`) behind a Plymouth splash.
`keyboard` comes before `autodetect`, so any keyboard works for the passphrase.

```sh
/etc/mkinitcpio.conf.d/arch.conf
--------------------------------
HOOKS=(base systemd plymouth keyboard autodetect microcode modconf kms sd-vconsole block sd-encrypt filesystems fsck)
```

Kernel parameters, with the UUID of the encrypted partition (`zswap` is off,
since it gets in the way of zram):

```sh
echo "rd.luks.name=$(blkid -s UUID -o value /dev/root_partition)=root root=/dev/mapper/root rootflags=subvol=@ rw quiet splash zswap.enabled=0" > /etc/kernel/cmdline
```

Rebuild the initramfs with these hooks:

```sh
mkinitcpio -P
```

### Bootloader

Limine, with a first boot entry. aarbs later hands the entries over to
limine-entry-tool (kernel updates) and limine-snapper-sync (snapshots in the
boot menu), and removes this one. Replace `X` with the drive as before:

```sh
mkdir -p /boot/EFI/limine
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/limine_x64.efi
efibootmgr --create --disk /dev/X --part 1 --label "Limine" --loader '\EFI\limine\limine_x64.efi' --unicode
```

```sh
cat > /boot/limine.conf <<EOF
timeout: 3

/Arch Linux (install)
    protocol: linux
    path: boot():/vmlinuz-linux
    cmdline: $(cat /etc/kernel/cmdline)
    module_path: boot():/initramfs-linux.img
EOF
cat /boot/limine.conf # check the cmdline
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

Enter the disk passphrase at the boot splash, then continue with the
[post-installation](post-installation.md).
