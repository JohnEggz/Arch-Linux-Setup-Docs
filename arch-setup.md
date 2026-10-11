# Arch Setup — History & Reference

> Machine: **t470-arch** — Lenovo ThinkPad T470
> Purpose: a chronological record of what was set up on this PC, **why** it was done,
> and **how** it was done, verified against the running system (not from memory).
> Small things are quoted inline; large/complex things point to the real files.

**Last verified:** 2026-10-11 (against the live system)

---

## 00 · Meta / conventions

### How to grep this document

Every entry carries tags on its own so you can filter the whole history:

```sh
grep -n 'status/todo'    arch-setup.md   # everything still to do
grep -n 'status/done'    arch-setup.md   # everything completed
grep -n 'status/partial' arch-setup.md   # started but not finished
grep -n '#inferred'      arch-setup.md   # rationale I reconstructed, not recorded at the time
grep -n '#verify'        arch-setup.md   # facts that still need root/manual confirmation
grep -n '#issue'         arch-setup.md   # known problems
grep -n '#cleanup'       arch-setup.md   # cleaning chores
grep -n '→'              arch-setup.md   # pointers to real config files
```

### Status legend

| Tag | Meaning |
|---|---|
| `#status/done` | Verified working on the live system |
| `#status/partial` | Works, but incomplete or not "riced" |
| `#status/todo` | Not done |
| `#status/blocked` | Cannot proceed without a decision/permission |
| `#inferred` | The "why" is reconstructed, not from an original note — correct me if wrong |
| `#verify/root` | Needs a root command to confirm |
| `#issue` | Known bug / failing thing |
| `#cleanup` | Leftover to tidy |

### Entry format

Each topic uses the same three headings:

- **What** — the outcome / current state
- **Why** — the reason (tagged `#inferred` when reconstructed)
- **How** — the commands and files that produced it

---

## 01 · Install (archinstall)

**What** — Base system installed with `archinstall`, using its guided configurator.
`#status/done`

**Why** — Fastest way to a working encrypted Arch base without hand-partitioning on the
first pass. `#inferred`

**How** — Run `archinstall` from the live ISO and select the options reflected below
(btrfs + encryption, pipewire, power-profiles-daemon, zsh, tmux, plymouth, limine,
reflector, ufw, NetworkManager). The result is what the rest of this document verifies.
No archinstall log was kept → the *recorded* source of truth is the running system.

---

## 02 · Disk layout, LUKS, btrfs subvolumes

**What** — Single NVMe, GPT, two partitions; LUKS2-encrypted btrfs with named subvolumes;
separate FAT32 ESP. `#status/done`

```
nvme0n1
├─ nvme0n1p1  vfat(FAT32)      1G   /boot            UUID 7B1D-1261   PARTUUID 3ffdc29b-a779-46b7-a0d3-def2066ffc4d
└─ nvme0n1p2  crypto_LUKS2          (LUKS)           UUID ef09fe3c-31f9-44ec-b45b-030153963c9b  PARTUUID db2ed609-e051-4d22-9082-c1f6eb2f9fc5
              └─ root  btrfs            UUID 25a8852d-2b8b-43fc-8d63-e2a2ab255db6
                 ├─ @              →  /
                 ├─ @home          →  /home
                 ├─ @pkg           →  /var/cache/pacman/pkg
                 └─ @log           →  /var/log
zram0        swap             4G   [SWAP]           UUID 7813b765-7ac4-45d3-aab0-e1ba979c5eec
```

**Why** — Full-disk encryption for a laptop; btrfs subvolumes so `/home`, pacman cache,
and logs can be snapshotted/separated (and excluded from snapshots) independently.
`=zstd:3` compression saves space on a small SSD. `#inferred`

**How** — Mount options per subvolume are in `/etc/fstab` (source of truth):

```
UUID=25a8852d-...  /                     btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@    0 0
UUID=25a8852d-...  /home                 btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@home 0 0
UUID=25a8852d-...  /var/cache/pacman/pkg btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@pkg  0 0
UUID=25a8852d-...  /var/log              btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@log  0 0
UUID=7B1D-1261     /boot                 vfat   rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro 0 2
```

- ESP is mounted **`/boot`** (not `/efi`), so the UKI and Limine live there.
- LUKS volume is unlocked early by the `encrypt` mkinitcpio hook via the kernel cmdline
  (see §03); there is **no** `/etc/crypttab` entry.
- Vendor: `btrfs-progs`.

---

## 03 · Boot: Limine (fallback path) + Unified Kernel Image + plymouth

**What** — Limine bootloader installed to the **EFI fallback path**, chainloading a
**Unified Kernel Image** (kernel + initramfs + cmdline in one `.efi`). Graphical boot via
plymouth. `#status/partial`

**Why** — Limine is simple and supports UKIs; a UKI keeps kernel, initramfs, and cmdline in
one signed-able file. Fallback path works without a custom NVRAM entry. `#inferred`

**How**

Boot menu — `/boot/EFI/BOOT/limine.conf` (source of truth):

```
/Arch Linux (linux)
    protocol: efi
    path: boot():/EFI/Linux/arch-linux.efi
    cmdline: cryptdevice=PARTUUID=db2ed609-e051-4d22-9082-c1f6eb2f9fc5:root root=/dev/mapper/root zswap.enabled=0 rootflags=subvol=@ rw rootfstype=btrfs quiet splash
```

UKI generation — `/etc/mkinitcpio.d/linux.preset`:

```
ALL_kver="/boot/vmlinuz-linux"
PRESETS=('default')                       # fallback preset disabled
default_uki="/boot/EFI/Linux/arch-linux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"
```

mkinitcpio hooks — `/etc/mkinitcpio.conf`:

```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block plymouth encrypt filesystems fsck)
```

- Relevant packages: `limine`, `mkinitcpio`, `intel-ucode`, `linux`, `linux-firmware`.
- The `limine` package ships **only EFI binaries** (`/usr/share/limine/BOOTX64.EFI` …) — there
  is **no entry tool**, so `limine.conf` and the fallback install were done **by hand**.
- Verified: `/boot/EFI/Linux/arch-linux.efi` is a PE32+ EFI application.

**Issues / notes** `#issue`
- Boot entry: `Boot0000* Limine` now exists on the current ESP (§16.2) and is first in
  `BootOrder`, in addition to the firmware `NVMe0` fallback path. Limine still loads the UKI
  via `limine.conf`.
- **Plymouth vs UKI splash overlap:** `plymouth` hook + `splash` cmdline + `--splash` BMP in the
  UKI. The Limine menu does not render a systemd-boot splash, so plymouth (theme `bgrt`) is the
  effective boot splash. Possible redundancy — revisit later. `#todo`
- Plymouth theme configured in `/etc/plymouth/plymouthd.conf`: `Theme=bgrt`.
- No boot-entry snapshots wired up (referenced in old todo as "pick snapshot on boot"). `#todo`

---

## 04 · Network + firewall

**What** — NetworkManager (with `wpa_supplicant`) manages Wi-Fi; UFW enabled with a
deny-incoming / allow-outgoing baseline. `#status/done`

**Why** — NM is the standard roaming-friendly manager; UFW gives a simple, sane default
firewall for a laptop. `#inferred`

**How**

- Packages: `networkmanager`, `wpa_supplicant`.
- Enabled services: `NetworkManager.service`, `NetworkManager-dispatcher.service`,
  `NetworkManager-wait-online.service`.
- Interfaces: `wlp4s0` (Intel Wireless 8265/8275), `enp0s31f6` (Ethernet), `lo`.
- UFW rules (from the original note; `#verify/root` re-check with `sudo ufw status verbose`):

```sh
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit 22/tcp
sudo ufw enable
sudo ufw status verbose
```

Expected: `ufw.service` enabled; `22/tcp LIMIT IN` (v4 + v6).

---

## 05 · Swap + power management

**What** — zram-based swap (4G, zstd) instead of a disk swap partition; power profiles via
power-profiles-daemon. `#status/done`

**Why** — zram swap is fast and avoids wearing the SSD / avoids an encrypted swap file
setup; PPD exposes the standard power modes. `#inferred`

**How**

- zram config `/etc/systemd/zram-generator.conf`:

```
[zram0]
compression-algorithm = zstd
```

- Active: `/dev/zram0` — 4G, replaces traditional swap. Package: `zram-generator`.
- Kernel cmdline sets `zswap.enabled=0` (zram is used instead of zswap — avoid double
  compression).
- Power: `power-profiles-daemon.service` enabled. Package: `power-profiles-daemon`.
  CPU driver `intel_pstate`. Profiles available: **performance**, balanced, power-saver.
  (Currently set to `performance` during setup.)
- Brightness keys handled by `brightnessctl` in Hyprland binds (§13).

---

## 06 · Audio (PipeWire)

**What** — Full PipeWire stack with WirePlumber session manager. `#status/done`

**Why** — PipeWire is the modern default on Arch; WirePlumber is its policy/session manager;
ALSA/JACK/Pulse shims cover legacy apps. `#inferred`

**How**

- Packages: `pipewire`, `pipewire-pulse`, `pipewire-alsa`, `pipewire-jack`,
  `wireplumber`, `gst-plugin-pipewire`, `libpulse` (client).
- User services enabled: `pipewire.socket`, `pipewire-pulse.socket`, `wireplumber.service`.
- Control via `wpctl` (used in Hyprland media-key binds).

---

## 07 · Bluetooth

**What** — BlueZ stack installed and enabled. `#status/done` `#issue`

**Why** — Laptop needs Bluetooth for peripherals/audio. `#inferred`

**How**

- Packages: `bluez`, `bluez-utils`.
- Enabled: `bluetooth.service`.
- Note: boot log shows `Bluetooth: hci0: Reading supported features failed (-16)` once —
  likely transient at cold boot; watch it. `#issue`

---

## 08 · Printing (CUPS)

**What** — CUPS installed with socket activation and a GUI config tool. `#status/done`

**Why** — Printing support; socket activation means cupsd starts only when needed. `#inferred`

**How**

- Packages: `cups`, `cups-pk-helper`, `system-config-printer`.
- Enabled: `cups.socket`, `cups.service`, `cups.path`.
- No printers configured yet. `#todo`

---

## 09 · Graphics / drivers

**What** — Intel graphics stack (Mesa, VA-API, Vulkan) working; `vkcube` verified. `#status/done`

**Why** — Kaby Lake iGPU; VA-API for hardware video decode, Vulkan for acceleration. `#inferred`

**How**

- Packages: `mesa`, `intel-media-driver`, `vulkan-intel`, `libva-utils`, `vulkan-tools`.
- GPU: Intel HD Graphics 620 (Kaby Lake-U GT2). CPU: Intel Core i5-7300U, 15 GiB RAM.
- Verified with `vainfo` and `vkcube` (both ran successfully during setup).

---

## 10 · Shell: zsh

**What** — zsh as login shell, with a custom two-line git-style prompt, completion engine,
history settings, and the fast-syntax-highlighting plugin. `#status/done`

**Why** — zsh + a fast syntax highlighter gives good interactive ergonomics.
`#inferred`

**How** — Files (source of truth): **→ `~/.zshrc`**, **→ `~/.zprofile`**

`~/.zprofile` (autostarts the Wayland session — see §13):

```sh
if uwsm check may-start && uwsm select; then
    exec uwsm start default
fi
```

`~/.zshrc` — substantive parts (full file is the source of truth):

```sh
export EDITOR="nvim"
export VISUAL="nvim"
export SUDO_EDITOR="nvim"

alias ..="cd .."

HISTFILE="$HOME/.zsh_history"
HISTSIZE=5000            # NOTE: original note said 50000 — actual value is 5000
SAVEHIST=5000
setopt SHARE_HISTORY
setopt EXTENDED_HISTORY
setopt HIST_IGNORE_DUPS
setopt HIST_REDUCE_BLANKS

# custom prompt: SSH indicator, colored path line, green/red ❯ cursor
autoload -U colors && colors
setopt PROMPT_SUBST
RPROMPT=""
precmd() { ... }         # see file

# completion engine
autoload -Uz compinit
compinit
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}' 'r:|=*' 'l:|=* r:|=*'
zstyle ':completion:*' menu select
zstyle ':completion:*' list-colors "${(s.:.)LS_COLORS}"

source ~/.zsh/plugins/fast-syntax-highlighting/fast-syntax-highlighting.plugin.zsh
```

- Plugin: **→ `~/.zsh/plugins/fast-syntax-highlighting`** (git clone of
  `https://github.com/zdharma-continuum/fast-syntax-highlighting.git`, commit `4672ad5`,
  dated 2026-08-31).
- `~/.bashrc` is left at the Arch default (color aliases only) — bash is not the login shell.
- Console keymap is `pl` (Polish); X11 layout unset (Hyprland uses `us`, §13).

---

## 11 · Fonts

**What** — Noto family + emoji + CJK, plus DejaVu and Liberation. `#status/done`

**Why** — Broad Unicode/emoji/CJK coverage and metric-compatible replacements for
Arial/Times/Courier. `#inferred`

**How** — Packages: `noto-fonts`, `noto-fonts-emoji`, `noto-fonts-cjk`, `ttf-dejavu`,
`ttf-liberation`. (~700 Noto faces registered.)

---

## 12 · Terminal: ghostty

**What** — Ghostty installed and used as the Hyprland terminal. `#status/partial` `#issue`

**Why** — Fast native Wayland terminal with modern features. `#inferred`

**How**

- Package: `ghostty`.
- Config dir **→ `~/.config/ghostty/`** — currently contains only an **empty, misnamed**
  `config.ghostty`. Ghostty expects `~/.config/ghostty/config`. Fix in §16.
- No visual customization yet ("no rice"). `#todo`

---

## 13 · Desktop: Hyprland + UWSM

**What** — Hyprland (Wayland compositor) launched via **UWSM**, using Hyprland's **new Lua
config format** (not the classic `hyprland.conf`). Lightly customized. `#status/partial`

**Why** — Hyprland for tiling; UWSM gives proper systemd-managed session/graphical-session
targets; Lua config is the modern format. `#inferred`

**How**

- Packages: `hyprland`, `uwsm`.
- Session files: `/usr/share/wayland-sessions/hyprland.desktop` and `hyprland-uwsm.desktop`.
- UWSM default **→ `~/.config/uwsm/default-id`** → `hyprland-uwsm.desktop`.
- Main config **→ `~/.config/hypr/hyprland.lua`** (autogenerated by Hyprland, then edited).
  Contenthighlights (this is a 12 KB file — read it directly):

  - Programs: `terminal = "ghostty"`. File manager is `yazi` (launched manually — no dedicated
    bind), app launcher not yet configured.
  - Autostart: `systemctl --user start hyprpolkitagent`.
  - Env: `XCURSOR_SIZE=24`, `HYPRCURSOR_SIZE=24`.
  - Look: `gaps_in=0`, `gaps_out=0`, `border_size=2`, rounded corners off (`rounding=0`),
    gradient active border, shadows + blur enabled, `layout="dwindle"`.
  - binds: `SUPER+RETURN` terminal, `SUPER+W` close, `SUPER+M` `hyprctl dispatch exit`,
    `SUPER+T` toggle float,
    `SUPER+SHIFT+B` librewolf, arrows = focus, `SUPER+[0-9]` workspaces, `SUPER+S` scratchpad
    `special:magic`, mouse drag/resize, 3-finger swipe = workspace.
  - Media keys: volume/mute/mic via `wpctl`; brightness via `brightnessctl -e4 -n2`;
    playback via `playerctl` (**not installed**, §17).
  - Input: `kb_layout="us"`, `follow_mouse=1`, natural scroll off.

**Issues / notes** `#issue`
- Binds referencing uninstalled binaries (`dolphin`, `hyprlauncher`) were **dropped**
  2026-10-11; `SUPER+M` simplified to `hyprctl dispatch exit`; `playerctl` installed so media
  keys work. `hyprland.lua` passes `luac -p`.
- No status bar / notification daemon / wallpaper / app launcher configured yet. `#todo`
  (`quickshell` is installed but unused — likely intended as the shell.)

---

## 14 · Session services (portals, polkit, clipboard)

**What** — Wayland desktop portal, polkit authentication agent, clipboard, and a mirror tool.
`#status/done`

**Why** — Portals let sandboxed apps (file picker, screenshare) work; polkit agent handles GUI
privilege prompts; wl-clipboard provides `wl-copy`/`wl-paste`. `#inferred`

**How**

- Packages: `xdg-desktop-portal-hyprland`, `hyprpolkitagent`, `wl-clipboard`, `wl-mirror`,
  `quickshell`.
- Portal backend: `/usr/share/xdg-desktop-portal/hyprland-portals.conf` →
  `[preferred] default=hyprland;gtk`.
- polkit agent started from Hyprland autostart (see §13); also runnable as
  `systemctl --user start hyprpolkitagent`.
- No user portal config override in `~/.config` (system default used).

---

## 15 · Applications

**What** — Browser, editor, monitor, file manager, and modern CLI tools. `#status/partial`

**Why** — Privacy browser + modal editor + TUI tools. `#inferred`

**How**

- Browser: `librewolf` (profile at **→ `~/.config/librewolf/librewolf/`**). No rice. `#todo`
- Editor: `neovim` (no config written yet). `#todo`
- System monitor: `btop` — config **→ `~/.config/btop/btop.conf`**.
- File manager: `yazi` (installed; Hyprland config still points at `dolphin`).
- CLI quality-of-life: `eza`, `bat`, `ripgrep`, `less`, `git`.
- Terminal multiplexer: `tmux` (no config file yet). `#todo`
- `opencode` installed (used to produce this document).

---

## 16 · Cleanup log

**What** — Housekeeping of leftovers from the current and previous installs.
`#status/partial`

**Why** — The machine has hosted several OSes (Windows, Ubuntu, GRUB, Omarchy) and their
stale EFI NVRAM entries remain; also pacman cache and minor config issues. `#inferred`

### 16.1 NVRAM boot entries (dead ones) `#cleanup` `#status/done`
Dead entries confirmed by `efibootmgr` (they point at partitions/dirs that no longer exist).
**All five were successfully removed** on 2026-10-11:

| Entry | Label | Pointed to | Result |
|---|---|---|---|
| `0000` | Windows Boot Manager | ESP `3ffdc29b…` `\EFI\Microsoft\…` (dir absent) | removed ✅ |
| `0010` | Ubuntu | ESP `82a51953…` (not present) | removed ✅ |
| `0012` | GRUB | ESP `e3d6c967…` (not present) | removed ✅ |
| `0013` | Omarchy | ESP `86d36399…` (old install) | removed ✅ |
| `0014` | Limine | ESP `86d36399…` (old install) | removed ✅ |

```sh
sudo efibootmgr -b 0000 -B
sudo efibootmgr -b 0010 -B
sudo efibootmgr -b 0012 -B
sudo efibootmgr -b 0013 -B
sudo efibootmgr -b 0014 -B
```

### 16.2 Boot entry for the current ESP `#cleanup` `#status/done`
Gave the current ESP a proper NVRAM entry instead of relying solely on the fallback path.
**Succeeded on retry (2026-10-11):** `Boot0000* Limine` →
`HD(1,GPT,3ffdc29b-…,0x800,0x200000)/\EFI\BOOT\BOOTX64.EFI`, and it is first in `BootOrder`.

```sh
sudo efibootmgr --create --disk /dev/nvme0n1 --part 1 --label Limine --loader '\EFI\BOOT\BOOTX64.EFI'
```

> Note: the first attempt (with a `\`-continued line inside a comment-bearing paste) appeared to
> succeed but did not persist — see §17 "Shell paste hazard". The clean single-line command
> above worked. Boot still ultimately loads the UKI via `limine.conf` (§03).

### 16.3 pacman cache `#cleanup` `#status/done`
Installed `pacman-contrib`, then pruned. **Cache 1.9 GB → 1.6 GB** (1223 → 1119 files).

```sh
sudo pacman -S --needed pacman-contrib
sudo paccache -rk1
sudo paccache -ruk0
```

> Remaining 1.6 GB = one (most recent) version of each installed package. To shrink further,
> `paccache -rk0` (keeps no cached versions) — not done, since keeping one rollback version is
> safer on a rolling-release system.

### 16.4 Ghostty config filename `#cleanup` `#status/done`
`~/.config/ghostty/config.ghostty` → `~/.config/ghostty/config` ✅ (done 2026-10-11).

### 16.5 Missing binaries referenced by Hyprland binds `#cleanup` `#status/done`
`playerctl` **installed** ✅ (2.4.1-5) — enables the media-key playback binds.
Per decision (2026-10-11): the **file-manager (`dolphin`) and launcher (`hyprlauncher`) binds
were dropped**; `SUPER+M` was simplified to `hyprctl dispatch exit` (dropped `hyprshutdown`).
`hyprland.lua` now has no references to uninstalled binaries and passes `luac -p`.

### 16.6 TPM services failing `#cleanup` `#issue` `#status/todo`
Three units fail on this T470 at boot: `systemd-tpm2-setup-early.service`,
`systemd-pcrproduct.service`, `systemd-pcrlogin@1000.service`, with
`Failed to initialize NvPCR index: No such file or directory`.

**Finding (2026-10-11):** the TPM is **already present and enabled** —
`/sys/class/tpm/tpm0`, `tpm_version_major = 2`, exposed via Intel PTT (`MSFT0101`). So this is
**not** a "TPM disabled in BIOS" problem. The units are failing because they want to use
**NvPCR** (non-volatile PCR), which this firmware TPM implementation does not provide.
No TPM-based disk unlock is configured (LUKS uses a passphrase), so these units are not needed.

**Decision (2026-10-11): mask the NvPCR units.** Run (root):

```sh
sudo systemctl mask systemd-tpm2-setup-early.service systemd-pcrproduct.service 'systemd-pcrlogin@*.service'
```

Then confirm no failures next boot: `systemctl --failed`.

### 16.7 Orphans / pacnew `#status/done`
None: `pacman -Qtdq` empty; no `*.pacnew`/`*.pacsave` under `/etc`.

---

## 17 · Known issues / discrepancies

- `#issue` **Old note vs reality** — the original `linux-notes.md` was partly inaccurate:
  - "**Termux**" was a typo for **tmux**.
  - `HISTSIZE 50000` → actual is **5000**.
  - "start-hyprland # ??" → actual launch is `uwsm start hyprland-uwsm.desktop` (via `~/.zprofile`).
  - Unrecorded but done: Bluetooth, CUPS, zram, fonts, neovim, git, yazi, quickshell,
    wl-mirror, brightnessctl, efibootmgr, btrfs-progs, intel-ucode, vulkan-tools, eza/bat/rg,
    gst-plugin-pipewire, wpa_supplicant.
- `#issue`~~`hyprland.lua` references uninstalled `dolphin`, `hyprlauncher`, `hyprshutdown`~~ — **resolved §16.5** (binds dropped; `playerctl` installed).
- `#issue`~~`~/.config/ghostty/config.ghostty` was empty and misnamed~~ — **resolved §16.4**.
- `#issue` TPM units failing at boot (§16.6).
- `#issue` **Shell paste hazard (lesson learned):** a paste block containing `#` comment lines
  was executed as commands by zsh — `(stop ...)` was read as a glob qualifier
  (`zsh: unknown sort specifier`) and `~1.9` as a `~user` expansion
  (`zsh: no such user or named directory: 1.9`). Remedy: command blocks in this document are
  now comment-free and avoid `~`/parens.
- `#issue` Unsure whether the pacman mirrorlist was actually regenerated by reflector —
  `#verify/root` `cat /etc/pacman.d/mirrorlist`.

---

## 18 · Todo

Deduplicated from the original note plus new items found during verification.

### Shell / CLI
- `#status/todo` zsh aliases for `eza`, `rg`, `bat` (packages installed, aliases not written).
- `#status/todo` review `less`, `head`, `tail` usage/aliases.
- `#status/todo` add `alias` for `l`/`ll` if desired.

### Editor
- `#status/todo` configure Neovim (`~/.config/nvim`).

### Terminal / multiplexer
- `#status/todo` Ghostty rice / real config.
- `#status/todo` tmux config (`~/.config/tmux/tmux.conf`).

### Desktop
- `#status/todo` Hyprland rice; decide on status bar / shell (`quickshell` installed).
- `#status/todo` notification daemon.
- `#status/todo` application launcher (`hyprlauncher`? `rofi`/`wofi`?).
- `#status/todo` wallpaper.
- `#status/todo` app audio control (per-app volume).
- `#status/todo` OSK (on-screen keyboard).
- `#status/todo` Hyprland plugins — evaluate Hyprspace vs Quickshell.

### System
- `#status/todo` backups strategy.
- `#status/todo` Limine: pick a snapshot on boot.
- `#status/todo` plymouth: review theme / redundancy with UKI splash.
- `#status/todo` AUR helper + verification workflow (no AUR packages yet).
- `#status/todo` mask the TPM NvPCR units — command pending, run as root (§16.6).
- `#status/todo` file manager — try something other than yazi.

---

## Appendix A — Explicit packages (verified)

`pacman -Qe`, 60 explicit packages (58 at install + `playerctl`, `pacman-contrib` added during cleanup), **no AUR/foreign packages**:

```
base bat bluez bluez-utils brightnessctl btop btrfs-progs cups cups-pk-helper
efibootmgr eza ghostty git gst-plugin-pipewire hyprland hyprpolkitagent
intel-media-driver intel-ucode less libpulse librewolf libva-utils limine linux
linux-firmware mkinitcpio neovim networkmanager noto-fonts noto-fonts-cjk
noto-fonts-emoji opencode pacman-contrib pipewire pipewire-alsa pipewire-jack
pipewire-pulse playerctl plymouth power-profiles-daemon quickshell reflector
ripgrep sudo system-config-printer tmux ttf-dejavu ttf-liberation ufw uwsm
vulkan-intel vulkan-tools wireplumber wl-clipboard wl-mirror wpa_supplicant
xdg-desktop-portal-hyprland yazi zram-generator zsh
```

## Appendix B — Enabled services (verified)

System:
```
bluetooth.service            cups.path / cups.service / cups.socket
getty@.service               NetworkManager.service
NetworkManager-dispatcher.service
NetworkManager-wait-online.service
power-profiles-daemon.service
reflector.timer              systemd-timesyncd.service
ufw.service                  systemd-userdbd.socket   remote-fs.target
```
User:
```
pipewire.socket  pipewire-pulse.socket  wireplumber.service  p11-kit-server.socket
```

## Appendix C — Key file pointers

| Path | What |
|---|---|
| `/etc/fstab` | Subvolume mounts + options |
| `/etc/mkinitcpio.conf` | Initramfs hooks (incl. plymouth, encrypt) |
| `/etc/mkinitcpio.d/linux.preset` | UKI generation settings |
| `/boot/EFI/BOOT/limine.conf` | Limine boot menu / cmdline |
| `/boot/EFI/Linux/arch-linux.efi` | Unified Kernel Image |
| `/etc/systemd/zram-generator.conf` | zram swap config |
| `/etc/plymouth/plymouthd.conf` | Boot splash theme (`bgrt`) |
| `/etc/xdg/reflector/reflector.conf` | Mirror selection settings |
| `~/.zshrc`, `~/.zprofile` | Shell config + session autostart |
| `~/.zsh/plugins/fast-syntax-highlighting` | zsh syntax plugin |
| `~/.config/hypr/hyprland.lua` | Hyprland config (Lua) |
| `~/.config/uwsm/default-id` | UWSM default session |
| `~/.config/btop/btop.conf` | btop config |
| `~/.config/ghostty/config` | Ghostty config |
| `~/.config/librewolf/librewolf/` | LibreWolf profiles |
