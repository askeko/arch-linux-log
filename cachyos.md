# CachyOS Optimizations

## Resources

[Optimized Repositories](https://wiki.cachyos.org/features/optimized_repos/)
[sched-ext](https://wiki.cachyos.org/configuration/sched-ext/)
[CachyOS Settings](https://github.com/CachyOS/CachyOS-Settings)

<!--toc:start-->
- [CachyOS Optimizations](#cachyos-optimizations)
  - [Resources](#resources)
  - [Repositories](#repositories)
  - [Settings](#settings)
  - [Kernel](#kernel)
  - [Scheduler](#scheduler)
  - [Process Priorities](#process-priorities)
  - [Gaming](#gaming)
  - [Undo](#undo)
<!--toc:end-->

Optional, done by hand after the [post-installation](post-installation.md), to
try out before anything goes into aarbs. snap-pac takes a snapshot before and
after every step, so each one can be undone (see
[Snapshots](post-installation.md#snapshots)).

## Repositories

Adds CachyOS' repositories above Arch's. The script picks the most optimized
build for the CPU (`x86-64-v3`, `x86-64-v4` or `znver4`) and backs up
`/etc/pacman.conf`. It also installs CachyOS' own pacman.

```sh
curl -LO https://mirror.cachyos.org/cachyos-repo.tar.xz
tar xvf cachyos-repo.tar.xz && cd cachyos-repo
sudo ./cachyos-repo.sh
```

Reinstall the repo packages to swap in the optimized builds (AUR packages stay
as they are):

```sh
pacman -Qqn | sudo pacman -S -
pacman -Qi glibc | grep -E 'Version|Architecture' # e.g. x86_64_v3
```

## Settings

CachyOS' defaults: sysctl and I/O scheduler tweaks, NTSYNC for Wine/Proton,
journal size, NVIDIA and audio power options. Ours win where they overlap:
`/etc/systemd/zram-generator.conf` and `99-vm-zram-parameters.conf` override
its zram and swappiness.

It also switches NetworkManager to systemd-resolved, which isn't running here
(no DNS!). Keep the current DNS setup, which wg-quick uses:

```sh
/etc/NetworkManager/conf.d/dns.conf
-----------------------------------
[main]
dns=default
```

```sh
sudo pacman -S cachyos-settings
lsmod | grep ntsync # after a reboot
```

## Kernel

`linux-cachyos` (BORE scheduler, more optimizations). limine-entry-tool adds it
to the boot menu; the stock `linux` stays as a fallback entry. Make it the
default by adding this line to `/etc/default/limine`:

```sh
/etc/default/limine
------------------
BOOT_ORDER="linux-cachyos, *, *fallback, Snapshots"
```

```sh
sudo pacman -S linux-cachyos linux-cachyos-headers
sudo pacman -S linux-cachyos-nvidia-open # lazarus only (NVIDIA)
sudo limine-update
```

Reboot, then check with `uname -r`.

## Scheduler

sched_ext runs a scheduler as a BPF program; `scx_lavd` is built for gaming and
desktop latency. If it crashes, the kernel falls back to its own scheduler.

```sh
/etc/scx_loader/config.toml
---------------------------
default_sched = "scx_lavd"
default_mode = "Auto"
```

```sh
sudo pacman -S scx-scheds scx-tools
sudo systemctl enable --now scx_loader.service
cat /sys/kernel/sched_ext/root/ops # scx_lavd
```

Try another one with `scxctl switch --sched bpfland`, stop with
`sudo systemctl disable --now scx_loader.service`.

## Process Priorities

ananicy-cpp renices known programs (games up, compilers down) with CachyOS'
rules. If the scheduler stalls or crashes, disable this first.

```sh
sudo pacman -S ananicy-cpp cachyos-ananicy-rules
sudo systemctl enable --now ananicy-cpp.service
```

## Gaming

Proton-CachyOS: select it per game (Properties → Compatibility) or as default
(Settings → Compatibility). `game-performance` (from cachyos-settings) switches
to the performance power profile while a game runs: set the launch options to
`game-performance %command%`.

```sh
sudo pacman -S proton-cachyos-slr power-profiles-daemon
sudo systemctl enable --now power-profiles-daemon.service
```

## Undo

Remove the repositories (restores `pacman.conf` and downgrades to Arch's
builds), then the rest:

```sh
cd cachyos-repo && sudo ./cachyos-repo.sh --remove
sudo pacman -Rns linux-cachyos linux-cachyos-headers cachyos-settings ananicy-cpp cachyos-ananicy-rules proton-cachyos-slr
```

On lazarus, also remove `linux-cachyos-nvidia-open`. Remove the `BOOT_ORDER`
line from `/etc/default/limine` and run `sudo limine-update`. Or restore the snapshot
from before the [Repositories](#repositories) step.
