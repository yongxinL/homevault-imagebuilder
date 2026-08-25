# HomeVault Imagebuilder

Ansible-based build system that produces a bootable Ubuntu live ISO for the HomeVault
home server / NAS appliance.  The image runs entirely from RAM with a read-only squashfs
base and writable overlayfs layers, and is designed to run inside a VMware ESXi 7.0 VM.

Copyright (C) 2026  George Li <yongxinl@outlook.com>

## Features

| Feature | Detail |
|---------|--------|
| Target OS | Ubuntu 24.04 LTS (Noble Numbat) — kernel 6.8.0-136-generic |
| Boot firmware | Hybrid BIOS (syslinux/isolinux) **and** UEFI (GRUB EFI) |
| Filesystem | Read-only squashfs + kernel overlayfs writable layers |
| Network | Netplan + systemd-networkd, DHCP with configurable timeout |
| NTP | chrony (replaces ntp/ntpdate) |
| Services | OpenSSH, Samba, NFSv4, Git, Ansible Core, Docker CE, Node.js |
| Web stack | Nginx, PHP-FPM 8.3 (ondrej/php PPA), MariaDB 11.4 LTS, Webmin 2.x |
| Container | Docker CE + docker-compose-plugin (Compose v2) |
| Automation | Ansible Core 2.20 (pip3 installed) |
| Python | python3 only — bundled with Ubuntu Noble, no separate installation |
| APT proxy | apt-cacher-ng compatible (optional, removed from final image) |
| Mirror | au.archive.ubuntu.com (configurable, Australian CDN by default) |

---

## Build Pipeline

The image is built in four sequential steps:

```
Step 1:  ansible-playbook -i hosts.yml build_initrd.yml
         → produces  /tmp/in_target/image/casper/vmlinuz
                     /tmp/in_target/image/casper/initrd.lz

Step 2:  ansible-playbook -i hosts.yml build_basefs.yml
         → produces  /tmp/in_target/image/casper/basefs.squashfs

Step 3:  ansible-playbook -i hosts.yml build_overlayfs.yml
         → produces  /tmp/in_target/image/casper/filesystem.squashfs

Step 4:  ansible-playbook -i hosts.yml build_image.yml
         → produces  /tmp/<hostname>_ubuntu_noble.iso
```

Each step must complete successfully before the next step can run.
Steps 1 and 2 can be run in parallel (they use separate chroot environments).

### Run the Playbooks as Root

**Recommendation:** execute `ansible-playbook` as the **root** user.  Many tasks
require root privileges and the playbooks do not use `become`, for example:

- the `chroot` connection plugin (all in-target tasks run inside `/opt/in_target/build/rootfs`)
- `mount` / `umount` of squashfs and overlayfs layers
- `debootstrap` and package installation inside the chroot
- `mksquashfs` ownership preservation

If the root account is locked (no password / no direct login) on your build host,
a sudoer can obtain a full root login shell first:

```bash
sudo -i
# then run the ansible-playbook commands below as root
```

### Build Host Requirements

- Ubuntu 22.04 or 24.04 (amd64)
- Ansible 2.15 or later
- root access — run `ansible-playbook` as root (see [Run the Playbooks as Root](#run-the-playbooks-as-root)); if the root account is locked, use `sudo -i` first
- Internet access or a configured apt-cacher-ng proxy

### Install Dependencies on the Build Host

```bash
sudo apt-get install -y ansible debootstrap squashfs-tools xorriso \
     grub-efi-amd64-bin grub-pc-bin mtools dosfstools
```

---

## Key Variables

All variables are defined in `group_vars/in_target.yml`.  Edit this file to
customise the image.  Per-phase variable overrides live in `vars/basefs.yml`,
`vars/initrd.yml`, `vars/image.yml`, and `vars/overlayfs.yml`.

### Identity

| Variable | Default | Description |
|----------|---------|-------------|
| `target_host_name` | `captainL` | Hostname written into the image and the boot menu |
| `target_host_domain` | `onelijia.com` | Domain name (used in /etc/hosts and SSH) |
| `target_root_password` | `secret` | Root password (plain-text; hashed with SHA-512 during build) |
| `target_timezone` | `Australia/Sydney` | IANA timezone string |
| `target_ntpserver` | `au.pool.ntp.org` | NTP pool base name for chrony |
| `target_dns_resolver` | `8.8.8.8` | Build-time DNS resolver (in /etc/resolv.conf during build only) |

### Kernel

| Variable | Default | Description |
|----------|---------|-------------|
| `build_kernel_version` | `6.8.0-136-generic` | **Exact** kernel version to install. Never use `linux-image-generic` — it pulls HWE meta-packages and bloats the image. See group_vars/in_target.yml for version table. |
| `debstrap_suite` | `noble` | Ubuntu release codename: `noble`, `jammy`, `focal`, `xenial` |
| `debstrap_arch` | `amd64` | Target CPU architecture |

### Mirror and Proxy

| Variable | Default | Description |
|----------|---------|-------------|
| `debstrap_base_mirror` | `au.archive.ubuntu.com` | Ubuntu archive mirror hostname. Change to your nearest mirror. |
| `apt_proxy` | `10.61.5.10:3142` | apt-cacher-ng proxy — format `host:port`. **Recommended** for faster repeat builds; see [Using an APT Proxy](#using-an-apt-proxy-apt-cacher-ng). Proxy is removed from the final image automatically. |

### Network

| Variable | Default | Description |
|----------|---------|-------------|
| `target_dhcp_timeout` | `30` | Seconds to wait for DHCP before continuing boot. Prevents hanging when no DHCP server is available. |
| `overylay_private_folder` | `/etc;/var/log;/home;/opt` | Semicolon-separated list of directories mounted as writable overlays at boot. |

### Storage

| Variable | Default | Description |
|----------|---------|-------------|
| `target_data_root` | `/oneLijia` | Root of the user data partition |
| `target_appdata_root` | `{{ target_data_root }}/private/var` | Application data directory (MariaDB, Docker data) |
| `target_swap_size` | `3g` | Swap file size created on the data partition at boot |
| `target_mount_device` | (list) | Disk-label → mount-point mappings for data partitions |

### Build Artefacts

| Variable | Default | Description |
|----------|---------|-------------|
| `build_image_iso` | `captainL_ubuntu_noble.iso` | Output ISO file name (auto-generated) |
| `build_base_image` | `basefs.squashfs` | Base filesystem image name |
| `build_overlay_image` | `filesystem.squashfs` | Overlay filesystem image name |

---

## Services Installed

### Base Filesystem (`build_basefs.yml`)

| Service | Version | Notes |
|---------|---------|-------|
| OpenSSH | Ubuntu default (~9.6) | Pre-generated host keys in role defaults |
| Samba | Ubuntu default (~4.19) | SMB/CIFS file sharing |
| NFSv4 | Ubuntu default | Network File System v4 |
| Git | Ubuntu default | Version control |
| Ansible Core | 2.20.x | Installed via pip3 |
| Docker CE | Latest stable | docker-compose-plugin (Compose v2) included |
| Node.js + pnpm | 24.x LTS / latest | Via NodeSource repository; pnpm via corepack (npm fallback) |
| open-vm-tools | Ubuntu default | VMware guest tools |
| chrony | Ubuntu default | NTP client (replaces ntp/ntpdate) |
| netplan.io | Ubuntu default | Network configuration |

### Overlay Filesystem (`build_overlayfs.yml`)

| Service | Version | Notes |
|---------|---------|-------|
| Nginx | Ubuntu default (~1.24) | Web server |
| PHP-FPM | 8.3 | Via ondrej/php PPA; full extension set |
| MariaDB | 11.4 LTS | Via MariaDB Foundation repo (AU mirror) |
| Webmin | 2.x | Via official Webmin repository |

---

## Boot Menu

Both the BIOS (syslinux) and UEFI (GRUB) menus present the same entries:

| Entry | Description |
|-------|-------------|
| **USR-A** (default) | Normal live boot — overlayfs and swap enabled |
| **USR-B** | Debug boot — `disable_overlay=yes`, no writable overlays |
| Boot local disk | Chain-boot the first local disk |
| Reboot / Power Off | System control |

### Boot Parameters (USR-A)

```
boot=casper hostname=<name> initrd=/casper/initrd.lz ip=frommedia
readonlydriver="/etc;/var/log;/home;/opt" swapfilesize=3g
```

---

## Data Partition Layout

The image mounts data partitions by filesystem label at boot time
(configured via `target_mount_device` in `group_vars/in_target.yml`):

| Label | Mount Point | Purpose |
|-------|-------------|---------|
| `dat01` | `/oneLijia` | Primary data partition |
| `vmf01` | `/oneLijia/vmfs` | VMware datastore |
| `dwn01` | `/oneLijia/downloading` | Downloads |
| `arc01` | `/oneLijia/archive` | Archive storage |

---

## Changes from Ubuntu 16 (Xenial)

### Breaking Changes

| Area | Ubuntu 16 | Ubuntu 24 |
|------|-----------|-----------|
| Network | `ifupdown` / `/etc/network/interfaces` | `netplan.io` + systemd-networkd |
| NTP | `ntp` + `ntpdate` packages | `chrony` |
| Live boot | `casper` + `lupin-casper` | `casper` only (`lupin-casper` removed in Ubuntu 20+) |
| Kernel overlayfs | `aufs-tools` | Kernel built-in overlayfs (since Linux 4.8; `aufs-tools` removed in Ubuntu 20+) |
| Python | `python` (2.7) + `python-apt` | `python3` + `python3-apt` (Python 2 not in Noble) |
| Docker Compose | standalone `docker-compose` v1 (pip) | `docker-compose-plugin` v2 (apt) |
| Docker repo | `apt.dockerproject.org` | `download.docker.com` |
| Ansible install | PPA `ppa:ansible/ansible` | pip3 (`ansible-core`) |
| MariaDB | 10.1 (`mariadb10` role) | 11.4 LTS (`mariadb11` role) |
| PHP | 8.2 | 8.3 (via ondrej/php PPA) |
| Node.js | 6.x | 24.x LTS |
| ISO tool | `genisoimage` (BIOS only) | `xorriso` (hybrid BIOS + UEFI) |
| Boot firmware | BIOS only | Hybrid BIOS + UEFI |
| Kernel packages | `linux-image-extra-*` separate package | Merged into `linux-image-*` in Ubuntu 20+ |
| Filesystem tools | `btrfs-tools` | `btrfs-progs` (renamed in Ubuntu 18+) |
| ExFAT | `exfat-fuse` + `exfat-utils` | `exfat-fuse` only (merged in Ubuntu 22+) |

### Removed Packages (no replacement needed)

| Package | Reason |
|---------|--------|
| `aufs-tools` | aufs kernel module removed in Ubuntu 20+; kernel overlayfs is built-in |
| `lupin-casper` | Removed upstream in Ubuntu 20+; standard casper covers all use cases |
| `isc-dhcp-client` | DHCP handled by systemd-networkd via netplan |
| `ifupdown` | Replaced by netplan.io |
| `ntp` / `ntpdate` | Replaced by chrony |
| `openssh-blacklist` / `openssh-blacklist-extra` | Removed in Ubuntu 20+; modern OpenSSH validates key strength natively |
| `python` / `python-minimal` / `python2.7*` | Python 2 not in Ubuntu 22+; Webmin 2.x uses Perl only |
| `ffmpeg2theora` | Unmaintained; not in Ubuntu 22+ repos. Use `ffmpeg -c:v libtheora` |
| `php-mongo` | Deprecated PHP MongoDB extension; replaced by `php-mongodb` |

---

## Ubuntu 22.04 (Jammy) Notes

To build a **Ubuntu 22.x** image instead of 24.x, make the following changes
in `group_vars/in_target.yml`:

```yaml
# 1. Change the Ubuntu release
debstrap_suite: 'jammy'

# 2. Change the kernel version (browse https://au.archive.ubuntu.com/ubuntu/pool/main/l/linux/)
build_kernel_version: '5.15.0-158-generic'
```

Additional manual changes required:

| File | Change needed |
|------|--------------|
| `roles/common/overlayfs_wrapper.yml` | Change `mariadb11` to `mariadb10` (MariaDB 11.4 requires Noble; Jammy uses 10.11) |
| `roles/services/mariadb10/defaults/main.yml` | Update `mariadb_version` to `10.11` and update repo key/URL to use `signed-by` approach |
| `roles/services/ansible/defaults/main.yml` | Jammy ships `ansible-core 2.12` in universe; pip3 approach still works for 2.20 — no change needed |
| `roles/services/nodejs/defaults/main.yml` | `nodejs_version: '24.x'` works on Jammy via NodeSource `nodistro` repo — no change needed |
| `roles/services/phpfpm8/defaults/main.yml` | ondrej/php PPA supports Jammy — no change needed |

> **Note:** The `mariadb10` role uses the deprecated `apt_key` module.
> For a production Jammy build, update the mariadb10 install task to use
> the `signed-by` keyring approach as done in `mariadb11`.

---

## Using an APT Proxy (apt-cacher-ng)

**Recommended:** run an apt-cacher-ng cache for builds — it speeds up repeated
builds substantially and saves bandwidth.  Set `apt_proxy` in
`group_vars/in_target.yml` to point at your cache:

```yaml
apt_proxy: '127.0.0.1:3142'
```

The proxy prefix is automatically prepended to all mirror URLs during the build.
The cleanup task removes the proxy configuration from the final image so the
live system fetches packages directly from the archive.

### Run apt-cacher-ng in Docker (recommended)

On a Docker host, start the cache using the `sameersbn/apt-cacher-ng` image:

```bash
docker run --name apt-cacher-ng --init -d --restart=always \
  --publish 3142:3142 \
  --volume /opt/containerd/apt-cacher-ng:/var/cache/apt-cacher-ng \
  sameersbn/apt-cacher-ng:latest
```

> **Note:** update the `--volume` host path (`/opt/containerd/apt-cacher-ng`) to a
> valid folder on your Docker host — this is where downloaded .deb packages are
> cached and persist across container restarts.
>
> Then point `apt_proxy` at that host, e.g. `apt_proxy: '<docker-host-ip>:3142'`.
> Image page: <https://hub.docker.com/r/sameersbn/apt-cacher-ng>

### Install apt-cacher-ng directly on the build host

Alternatively, install it natively on the build host:

```bash
sudo apt-get install -y apt-cacher-ng
sudo systemctl enable --now apt-cacher-ng
```

---

## Disk Installer

After booting from the ISO, run the automated disk installer:

```bash
sudo bash /cdrom/syslinux/install.sh
```

This creates a GPT partition table on `/dev/sda`:
- Partition 1 — BIOS boot (1 MB, bios_grub flag)
- Partition 2 — EFI System Partition (512 MB, FAT32, ESP flag)
- Partition 3 — Data partition (remaining space, ext4, label `dat01`)

To set a custom hostname during install:

```bash
sudo newName=myserver bash /cdrom/syslinux/install.sh
```

---

## Project Structure

```
homevault-imagebuilder/
├── build_initrd.yml          # Phase 1: kernel + initrd
├── build_basefs.yml          # Phase 2: base filesystem
├── build_overlayfs.yml       # Phase 3: overlay (app services)
├── build_image.yml           # Phase 4: ISO assembly
├── hosts.yml                 # Ansible inventory (YAML format)
├── group_vars/
│   └── in_target.yml         # ← Primary configuration — edit this
├── vars/
│   ├── basefs.yml            # Phase 2 package lists
│   ├── initrd.yml            # Phase 1 package lists
│   ├── image.yml             # Phase 4 settings
│   └── overlayfs.yml         # Phase 3 settings
└── roles/
    ├── common/               # Shared mount/unmount/cleanup tasks
    ├── basefs/               # Base filesystem build stages
    ├── initrd/               # Kernel/initrd build stages
    ├── overlayfs/            # Overlay filesystem build stages
    ├── imgbuild/             # ISO assembly stages
    ├── hardware/
    │   ├── vmtools/          # VMware open-vm-tools
    └── services/
        ├── users/            # User accounts and groups
        ├── openssh/          # SSH server
        ├── samba/            # Samba SMB/CIFS
        ├── nfsv4/            # NFS v4 server
        ├── git/              # Git
        ├── ansible/          # Ansible Core 2.20 (pip3)
        ├── docker/           # Docker CE + Compose v2
        ├── nodejs/           # Node.js 24.x LTS
        ├── nginx/            # Nginx web server
        ├── phpfpm8/          # PHP-FPM 8.3
        ├── mariadb10/        # MariaDB 10.x (Ubuntu 22 / legacy)
        ├── mariadb11/        # MariaDB 11.4 LTS (Ubuntu 24 — default)
        ├── webmin/           # Webmin 2.x
        └── certificate/      # SSL/TLS certificate generation
```
