# DrivesManager — btrfs + ntfs Automount Sidebar Fix (AppImage)

Single-file GUI + CLI that makes permanent `/mnt` data drives show up in **Nautilus** and **PCManFM-Qt (Lubuntu/LXQt)** sidebars — not just Dolphin — and keeps the list limited to **btrfs + ntfs** where it belongs.

## The problem

Drives mounted under `/mnt/...` via `/etc/fstab` without `x-gvfs-show`:

- **Dolphin** shows them (it reads `/proc/mounts` directly).
- **Nautilus / PCManFM-Qt** hide them (GVFS/udisks2 treats `/mnt` as system paths unless `x-gvfs-show` is present).

So the same machine shows different drives depending on which file manager you open. This tool fixes that at the source: the fstab options.

## What it does

- **`--list`** — lists btrfs + ntfs data drives only (`/dev/sda7`, `/dev/sda8`, `/dev/sda9`, `/dev/sda10` on a typical box). System members (`/`, `/home`, `/swap`, `[SWAP]`, `/boot/efi`), `vfat`, `exfat`, `swap`, `squashfs`, and `loop` devices are excluded. No whitespace-parsing hacks: `lsblk --pairs` output is parsed, so empty `LABEL` fields can't shift columns.
- **`--fix`** — idempotent. Backs up `/etc/fstab`, dedupes the swap line, merges duplicate `# >>> automount (managed) >>>` blocks into one, and adds `x-gvfs-show,x-gvfs-name=<pretty>` to every `/mnt` entry. Sidebar names are fstab-safe (`ESP-EAC4-0450`, `DATA-D3F0-737C`, `Linux-558ae5c5`, `Win-34E6BEE2`, `RECOVERY`, `Data250G-Top/Root/Home`).
- **`--prune`** — idempotent. Drops non-btrfs/ntfs lines (e.g. leftover `vfat`) from the managed block. Keeps 6 entries: 4× btrfs + 2× ntfs.
- **GUI** — drive table, *Open in Files* (xdg-open → pcmanfm-qt → nautilus → dolphin), and *Fix sidebar* which runs `--fix --prune` elevated and reloads systemd. Verification is live: `gio mount -l` shows the `GProxyVolume` entries immediately, no reboot required in most cases.

## Files

| File | Purpose |
|---|---|
| `DrivesManager-x86_64.AppImage` | Portable app, icon bundled, runs anywhere on x86_64 |
| `drives-manager-rs/` | Rust source (std-only, zero crates). Build: `cargo build --release` |
| `drives-manager-rs/target/release/drives-manager-rs` | Standalone CLI if you don't want the AppImage |

## Usage

Double-click the AppImage, or:

```bash
./DrivesManager-x86_64.AppImage            # GUI
./DrivesManager-x86_64.AppImage --list     # via AppRun passthrough (or use the Rust binary)
```

CLI (needs root for fstab writes):

```bash
pkexec ./drives-manager-rs --fix --prune
sudo umount /mnt/disk-eac4-045 /mnt/disk-d3f0-737   # drop the retired vfat mounts once
sudo systemctl daemon-reload
gio mount -l | grep -E "Volume|Mount\(0\)"          # verify sidebar entries
```

Every fstab write creates a timestamped backup (`/etc/fstab.fix-bak-*`, `/etc/fstab.prune-bak-*`) and every line keeps `nofail,x-systemd.device-timeout=15`, so a missing disk can never hang the boot. `findmnt --verify` should report `0 parse errors, 0 errors` afterwards (ntfs-3g FUSE warnings are expected noise).

## Why Rust, why std-only

The fix logic used to be throwaway Python. It worked until it didn't: `lsblk` text parsing broke on empty labels, and the AppImage + `pkexec` combination failed with `Errno 13` (see below). Rewriting in Rust with zero dependencies gives one static binary, `cargo test` coverage for the parsers, and no interpreter drift between machines.

## Gotcha worth knowing: AppImage FUSE mounts are invisible to root

An AppImage runs from a user-owned FUSE mount (`/tmp/.mount_XXXXXX`). `pkexec` switches to root, and root **cannot read that mount** (FUSE `user_allow_other` is off by default). So `pkexec python3 /tmp/.mount_.../fix.py` dies with `Permission denied` even though the file is right there.

The fix used here, in both the GUI and `apply-fix.sh`:

1. Prefer a helper binary installed **outside** the mount (`~/Applications/DrivesManager-files/drives-manager-rs`).
2. If only the in-mount copy exists, stage a world-readable copy to `/tmp/drives-mgr-*` (`0755` for binaries, `0644` for scripts) and `pkexec` *that*.

If you ship any AppImage that elevates, handle this or your users will file exactly this bug.

## Requirements

- x86_64 Linux (tested on Lubuntu/LXQt with Nautilus + PCManFM-Qt + Dolphin installed)
- `lsblk`, `pkexec` (or `sudo`), `systemctl`, `zenity` (Rust `--help`-less GUI mode), `findmnt` for verification
- No Python needed at runtime for the Rust path

## Safety

- Timestamped fstab backup before every mutation. Restore: `sudo cp /etc/fstab.prune-bak-<ts> /etc/fstab && sudo systemctl daemon-reload`.
- All managed lines carry `nofail` — boot never blocks on a missing drive.
- The tool only touches lines inside `# >>> automount (managed) >>>` markers plus stray `/mnt` lines; system mounts (`/`, `/home`, `/boot/efi`, swap) are never rewritten.
