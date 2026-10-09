# Post installation

[Installation Guide]: installation.md
[aarbs]: https://github.com/askeko/aarbs
[absrice]: https://github.com/askeko/absrice
[gh auth login]: https://cli.github.com/manual/gh_auth_login
[rbw]: https://github.com/doy/rbw#configuration
[Hyprland Screen-Sharing]: https://wiki.hypr.land/Useful-Utilities/Screen-Sharing/
[systemd-cryptenroll]: https://wiki.archlinux.org/title/Systemd-cryptenroll
[Snapper]: https://wiki.archlinux.org/title/Snapper

<!--toc:start-->
- [Post installation](#post-installation)
  - [Install Script](#install-script)
    - [Testing Unpublished Changes](#testing-unpublished-changes)
  - [First Login](#first-login)
  - [Security](#security)
    - [YubiKey Disk Unlock](#yubikey-disk-unlock)
  - [Accounts](#accounts)
    - [GitHub](#github)
    - [Claude Code and Codex](#claude-code-and-codex)
    - [Bitwarden (rbw)](#bitwarden-rbw)
  - [Programs](#programs)
    - [Firefox](#firefox)
    - [Neovim](#neovim)
  - [WireGuard](#wireguard)
  - [Firewall](#firewall)
  - [Snapshots](#snapshots)
    - [Restore a Snapshot](#restore-a-snapshot)
  - [Updating](#updating)
  - [Troubleshooting](#troubleshooting)
  - [Desktop Specific (lazarus)](#desktop-specific-lazarus)
    - [Data SSD](#data-ssd)
<!--toc:end-->

## Install Script

After following the steps in the [installation guide][Installation Guide] and
logging in as root (connect Wi-Fi with `nmtui`), download [aarbs] and its
package list:

```sh
curl -LO https://raw.githubusercontent.com/askeko/aarbs/main/aarbs.sh
curl -LO https://raw.githubusercontent.com/askeko/aarbs/main/progs.csv
```

Run script:

```sh
sh aarbs.sh
```

It installs the GPU drivers and programs, creates the user, applies [absrice]
with chezmoi (no need to import dotfiles), and sets up greetd, services, zram,
snapshots and the Limine boot entries.
If it fails, check `/var/log/aarbs.log`, fix the cause and run it again.
Reboot when it's done.

### Testing Unpublished Changes

aarbs clones the dotfiles from GitHub. To test local changes in a VM, copy
`aarbs.sh` and `progs.csv` over, commit the absrice changes to a copy in
`/tmp/absrice-test`, and point `dotfilesrepo` in `aarbs.sh` at that path.

## First Login

After the disk is unlocked you are logged in automatically (tuigreet only
shows after a logout). `Mod+Shift+/` lists all keybinds.

The first app that stores a secret (gh, Discord, Obsidian) asks to create the
default keyring. Leave its password empty: the disk is already encrypted, and
with autologin a password would mean an unlock prompt on every boot.

Wallpapers go in `~/pictures/wallpapers` (`Mod+B` to pick one). Downloads,
documents, pictures etc. all point to `~/tmp`, which is cleaned after 7 days.

## Security

### YubiKey Disk Unlock

Enroll each YubiKey (touch it when it blinks), then a recovery key. Write the
recovery key down and keep it away from the computer ([systemd-cryptenroll]).

```sh
lsblk -f # the crypto_LUKS partition is /dev/root_partition
sudo systemd-cryptenroll --fido2-device=auto /dev/root_partition # once per key
sudo systemd-cryptenroll --recovery-key /dev/root_partition
sudo cryptsetup open --test-passphrase /dev/root_partition # type the stored recovery key
```

Keep a typed secret: an update that breaks FIDO2 unlock locks out both keys.
To drop the everyday passphrase once the recovery key is stored:
`sudo systemd-cryptenroll --wipe-slot=password /dev/root_partition`.

Add `rd.luks.options=LUKS_UUID=fido2-device=auto` to `/etc/kernel/cmdline`,
with the same UUID as in `rd.luks.name`, then update the boot entries:

```sh
sudo nvim /etc/kernel/cmdline
sudo limine-update
```

At boot, touch the key (and enter its PIN if it has one). Without the key, the
passphrase prompt appears after 30 seconds.

### YubiKey Login

Every login (TTY, tuigreet, lock screen) asks for the password, then a touch on
a YubiKey (the key blinks; hyprlock shows no prompt). Until the mapping file
exists the password alone logs in. Pulling out a key locks the screen, so
register both keys into a temporary file (the swap locks; unlock with the
password) and move it into place last:

```sh
mkdir -p ~/.config/Yubico
pamu2fcfg    > ~/.config/Yubico/u2f_keys.new # first key, touch when it blinks
pamu2fcfg -n >> ~/.config/Yubico/u2f_keys.new # second key
mv ~/.config/Yubico/u2f_keys.new ~/.config/Yubico/u2f_keys
```

Stuck lock screen: `Ctrl+Alt+F2`, log in (password + touch), `pkill -USR1 hyprlock`.

Both keys lost: boot the Arch ISO, unlock with the recovery key, delete the
mapping file:

```sh
cryptsetup open /dev/root_partition root
mount -o subvol=@home /dev/mapper/root /mnt
rm /mnt/USER/.config/Yubico/u2f_keys
```

## Accounts

### GitHub

Log in and upload an SSH key ([gh auth login]):

```sh
gh auth login --hostname github.com --git-protocol ssh --web
```

Switch the dotfiles remote to SSH to push changes:

```sh
chezmoi git -- remote set-url origin git@github.com:askeko/absrice.git
```

### Claude Code and Codex

Both are installed into `~/.local/bin` when the dotfiles are applied. Run
`claude` and `codex` to sign in.

### Bitwarden (rbw)

Not tracked in the dotfiles (public repo). Set up once ([rbw]):

```sh
rbw config set email 'you@example.com'
rbw config set pinentry pinentry-rofi
rbw register # personal API key, for the official server
rbw login
```

`Mod+M` opens the password menu.

## Programs

### Firefox

Sign in to sync the extensions (Tree Style Tab). The dotfiles only hide the
tab bar.

### Neovim

The first start downloads LazyVim's plugins and Mason's language servers.
Check with `:checkhealth lazyvim` and `:Mason`.

### Nix

For programs needed once, and project dev shells. CLI tools only: GUI apps from
Nix can't find Arch's GPU drivers. Unused store paths are removed weekly.

```sh
nix shell nixpkgs#ffmpeg # gone when the shell exits
echo 'use flake' > .envrc && direnv allow # project with a flake.nix
```

## WireGuard

Profiles hold private keys, so they stay root-only:

```sh
# Replace home with the profile name (max 15 chars: letters, digits, _=+.-)
sudo install -m 600 path/to/profile.conf /etc/wireguard/home.conf
```

`Mod+Shift+V` connects/disconnects.

## Firewall

aarbs sets up an nftables firewall that drops incoming connections, except
replies, ping, DHCP/DNS for local VMs and Steam (Remote Play, dedicated
server). To open a port, add a rule to `/etc/nftables.conf`, then:

```sh
sudo nft -f /etc/nftables.conf
sudo nft list ruleset # docker and libvirt add their own tables
```

Docker's published ports bypass it: use `-p 127.0.0.1:8080:80` for
local-only services.

## Snapshots

[Snapper] snapshots `/` before and after every pacman transaction and keeps
the last 10. `/home` isn't snapshotted. limine-snapper-sync adds each one to the
boot menu (Snapshots) with its matching kernel.

```sh
snapper -c root list # snapshots and the pacman command behind them
snapper -c root status 41..42 # what changed between two snapshots
sudo snapper -c root undochange 41..42 # revert those changes
limine-snapper-info # bootable snapshots and EFI partition usage
```

### Restore a Snapshot

When the system is too broken for `undochange`, or doesn't boot: pick a
snapshot under Snapshots in the boot menu. It runs on a temporary overlay
(changes are lost at reboot). Then click "Restore now" in the notification, or:

```sh
sudo limine-snapper-restore
```

This replaces `@` with the snapshot and keeps the old system as a "backup"
entry in the boot menu, to go back or remove later.

## Updating

```sh
yay # repo and AUR packages
chezmoi update # pull and apply dotfile changes
```

yay shows what changed in each AUR package's build files (the whole thing for
new packages) before building. Read it, then answer "Proceed with install?".

Edit dotfiles with `chezmoi edit <file>`, then `chezmoi diff` and
`chezmoi apply`.

## Troubleshooting

```sh
hyprctl configerrors
systemctl --user --failed
journalctl --user -b -u waybar.service # or any other user service
sudo journalctl -b -u greetd.service
```

If the graphical login fails, switch to a TTY with `Ctrl+Alt+F2` and check the
logs there.

Screensharing: see [Hyprland screen-sharing][Hyprland Screen-Sharing] and
`journalctl --user -b -u xdg-desktop-portal-hyprland.service`.

## Desktop Specific (lazarus)

### Data SSD

The NTFS data drive (UUID from abslab: `5672622A72620F55`, verify with
`lsblk -f`). The kernel's ntfs3 driver needs no extra package.

```sh
sudo mkdir -p /mnt/data
```

```sh
/etc/fstab
-------------
UUID=5672622A72620F55 /mnt/data ntfs3 uid=1000,gid=1000,dmask=022,fmask=022,nofail 0 0
```

Check that `id` shows uid/gid 1000, then `sudo systemctl daemon-reload` and
`sudo mount /mnt/data`.
