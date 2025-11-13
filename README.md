# Arch Linux Installation Notes

Installed using the official **archinstall** script for a faster base setup. There are a lot of hate towards archinstall on the internet, but I tried it and found it to be a great tool for automating the installation process. I don't think, it's made for beginners though as I think Arch should be installed in an opinionated way. Installing whatever the defaults are without understanding the choices and its implications is not how Arch should be installed. It's important to take the time to understand the choices and their implications before making them. But for those who are already familiar with Arch Linux and have a good understanding of the system, archinstall can be a great tool for automating the installation process.

> [!NOTE]
> This guide is intended for personal reference and is not intended as a tutorial.

---

## Disk & Filesystem Configuration

### Disk Partitioning
Performed **manual partitioning** with the following layout:

| Mount Point | Size | Type | Filesystem | Description |
|--------------|------|------|-------------|--------------|
| `/efi` | 1 GB | EFI System Partition | vfat | Mounted at `/efi` instead of `/boot` (current Arch recommendation) |
| `/` | Rest of disk | LUKS Encrypted | btrfs | Main system volume |

Encryption was configured with **LUKS** on the root partition.
The decrypted device is mapped as `/dev/mapper/root`.

---

### Btrfs Configuration

- **Filesystem:** `btrfs`
- **Compression:** `zstd:3`
- **SSD-optimized mount options:** `ssd,space_cache=v2,relatime`
- **Special mount options:**
  - `/swap`, `/var`, `/var/log` use `nodatacow` to prevent CoW overhead.
- **Subvolume layout:** based on openSUSE’s recommended structure, with minor adjustments.

| Mount Point | Subvolume | Options | Notes |
|--------------|------------|----------|-------|
| `/` | `@` | compress=zstd:3 | Root filesystem |
| `/.snapshots` | `@snapshots` | compress=zstd:3 | System snapshots |
| `/boot` | `@boot` | compress=zstd:3 | Boot files on Btrfs |
| `/home` | `@home` | compress=zstd:3 | User home directory |
| `/home/.snapshots` | `@home_snapshots` | compress=zstd:3 | Home snapshots |
| `/opt` | `@opt` | compress=zstd:3 | Optional software |
| `/root` | `@root` | compress=zstd:3 | Root user’s home |
| `/srv` | `@srv` | compress=zstd:3 | Server data |
| `/swap` | `@swap` | nodatacow | Btrfs swap directory |
| `/tmp` | `@tmp` | compress=zstd:3 | Temporary files |
| `/usr/local` | `@usr_local` | compress=zstd:3 | Local programs |
| `/var` | `@var` | nodatacow | Logs, databases, caches |
| `/var/cache/pacman/pkg` | `@pkg` | compress=zstd:3 | Pacman cache |
| `/var/log` | `@log` | nodatacow | System logs |

---

### Example
```
# /etc/fstab

# ───────────────────────────────────────────────
# Btrfs subvolumes on LUKS root
# ───────────────────────────────────────────────
/dev/mapper/root  /                   btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@                 0  0
/dev/mapper/root  /.snapshots         btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@snapshots        0  0
/dev/mapper/root  /boot               btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@boot             0  0
/dev/mapper/root  /home               btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@home             0  0
/dev/mapper/root  /home/.snapshots    btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@home_snapshots    0  0
/dev/mapper/root  /opt                btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@opt              0  0
/dev/mapper/root  /root               btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@root             0  0
/dev/mapper/root  /srv                btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@srv              0  0
/dev/mapper/root  /swap               btrfs  rw,relatime,nodatacow,ssd,space_cache=v2,subvol=/@swap                   0  0
/dev/mapper/root  /tmp                btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@tmp              0  0
/dev/mapper/root  /usr/local          btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@usr_local        0  0
/dev/mapper/root  /var                btrfs  rw,relatime,nodatacow,ssd,space_cache=v2,subvol=/@var                    0  0
/dev/mapper/root  /var/cache/pacman/pkg  btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@pkg           0  0
/dev/mapper/root  /var/log            btrfs  rw,relatime,nodatacow,ssd,space_cache=v2,subvol=/@log                    0  0

# ───────────────────────────────────────────────
# EFI System Partition
# ───────────────────────────────────────────────
/dev/nvme0n1p1   /efi                vfat   rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro  0  2

# ───────────────────────────────────────────────
# Swapfile inside Btrfs subvolume
# ───────────────────────────────────────────────
/swap/swapfile   none                swap   defaults  0  0

```


---

## Swap Configuration

- **No dedicated swap partition**.
- Created **Btrfs-based swap file** under `/swap/swapfile`.
- Added to `/etc/fstab` after installation.
- Mounted with `nodatacow` for stability.
- **No hibernation configured** (swap size flexible and adjusted as needed).

**zram:** disabled during installation.
**zswap:** enabled later as the preferred compression-based swap optimization for 16 GB RAM systems.

---

## Bootloader

- **Bootloader:** `systemd-boot`
- **Unified Kernel Image (UKI):** enabled and used for systemd-boot integration.
- **Reasoning:**
  - Cleaner, modern, and aligns with systemd’s boot model.
  - `grub-btrfs` not used intentionally; snapshot booting not considered critical.
  - For system recovery, a live USB + `arch-chroot` workflow is preferred.

---

## Post-Installation Setup

- Skipped **Profiles** and **Additional Packages** in `archinstall`.
- Uses a **custom install script** to:
  - Install essential packages.
  - Configure the **niri** Wayland compositor.
  - Apply dotfiles from a personal repository.

The setup avoids desktop environments and uses a minimal, modular approach based on personal configuration.

---

## Snapper & Snapshot Management

### Overview

Btrfs snapshot management is handled through **Snapper** and **snap-pac**.

- **snapper** provides automatic timeline and pre/post snapshot functionality.
- **snap-pac** integrates with Pacman to automatically take snapshots before and after package transactions.
- Configured for both `/` (system) and `/home` (user data).
- Each subvolume was prepared for snapper manually to ensure clean integration with the existing Btrfs layout.

---

### Snapper Configuration Setup

Since `/` and `/home` subvolumes were already created during installation, Snapper needed to be reinitialized cleanly for both.

**Steps performed:**

  ```bash
  sudo rm -rf /.snapshots /home/.snapshots
  Created Snapper configurations for both system and home:

  sudo snapper -c root create-config /
  sudo snapper -c home create-config /home


  Deleted the automatically created /.snapshots and /home/.snapshots directories again:

  sudo rm -rf /.snapshots /home/.snapshots


  Recreated empty directories and remounted according to /etc/fstab:

  sudo mkdir /.snapshots /home/.snapshots
  sudo mount -a

```


  This ensured Snapper was using the correct, pre-defined subvolumes (@snapshots and @home_snapshots).

  Snapper Configuration Files

  Located in /etc/snapper/configs/:

  /etc/snapper/configs/root

  /etc/snapper/configs/home

  Typical adjustments made:

  Option	Value	Notes
  SUBVOLUME	/ and /home	Matches fstab layout
  ```
  TIMELINE_CREATE	yes	Enable automatic timeline snapshots
  TIMELINE_CLEANUP	yes	Allow periodic cleanup
  NUMBER_CLEANUP	yes	Maintain fixed number of snapshots
  TIMELINE_LIMIT_HOURLY	10	Keep up to 10 hourly snapshots
  TIMELINE_LIMIT_DAILY	7	Keep up to 7 daily snapshots
  TIMELINE_LIMIT_WEEKLY	0	Disabled
  TIMELINE_LIMIT_MONTHLY	0	Disabled
  TIMELINE_LIMIT_YEARLY	0	Disabled
  ```
