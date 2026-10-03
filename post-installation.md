# Post installation

[Installation Guide]: installation.md
[aarbs]: https://github.com/askeko/aarbs
[absrice]: https://github.com/askeko/absrice
[gh auth login]: https://cli.github.com/manual/gh_auth_login
[rbw]: https://github.com/doy/rbw#configuration
[Hyprland Screen-Sharing]: https://wiki.hypr.land/Useful-Utilities/Screen-Sharing/

<!--toc:start-->
- [Post installation](#post-installation)
  - [Install Script](#install-script)
    - [Testing Unpublished Changes](#testing-unpublished-changes)
  - [First Login](#first-login)
  - [Accounts](#accounts)
    - [GitHub](#github)
    - [Claude Code and Codex](#claude-code-and-codex)
    - [Bitwarden (rbw)](#bitwarden-rbw)
  - [Programs](#programs)
    - [Firefox](#firefox)
    - [Neovim](#neovim)
  - [WireGuard](#wireguard)
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
with chezmoi (no need to import dotfiles), and sets up greetd and services.
If it fails, check `/var/log/aarbs.log`, fix the cause and run it again.
Reboot when it's done.

### Testing Unpublished Changes

aarbs clones the dotfiles from GitHub. To test local changes in a VM, copy
`aarbs.sh` and `progs.csv` over, commit the absrice changes to a copy in
`/tmp/absrice-test`, and point `dotfilesrepo` in `aarbs.sh` at that path.

## First Login

Log in through tuigreet. `Mod+Shift+/` lists all keybinds.

Wallpapers go in `~/pictures/wallpapers` (`Mod+B` to pick one). Downloads,
documents, pictures etc. all point to `~/tmp`, which is cleaned after 7 days.

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

## WireGuard

Profiles hold private keys, so they stay root-only:

```sh
# Replace home with the profile name (max 15 chars: letters, digits, _=+.-)
sudo install -m 600 path/to/profile.conf /etc/wireguard/home.conf
```

`Mod+Shift+V` connects/disconnects.

## Updating

```sh
yay # repo and AUR packages
chezmoi update # pull and apply dotfile changes
```

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
