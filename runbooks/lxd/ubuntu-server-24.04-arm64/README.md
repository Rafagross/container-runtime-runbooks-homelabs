# Install LXD — Ubuntu Server 24.04 LTS (arm64)

**Author:** Rafael Gross
**Reviewed:** April 2026
**LXD Version:** 5.21 LTS (snap)
**Platform:** Ubuntu Server 24.04 LTS (Noble Numbat) · arm64 · VirtualBox VM

---

## Overview

LXD is a container and VM manager built on top of LXC. It provides a unified REST API-driven CLI for managing system containers and virtual machines — covering storage pools, bridged networking, snapshots, and remote instance management.

On Ubuntu 24.04, the APT-based `lxd` package no longer exists. Canonical removed it starting with Ubuntu 22.04 and now exclusively supports LXD via the **snap package**, which ships with native `arm64` support and is pre-installed on Ubuntu Server 24.04.

---

## Environment

| Property | Value |
|----------|-------|
| OS | Ubuntu Server 24.04 LTS (Noble Numbat) |
| Architecture | arm64 (aarch64) |
| Deployment | VirtualBox VM — Homelab |
| LXD Version | 5.21 LTS (current stable snap track) |
| Install Method | `sudo snap install lxd` |
| snapd | Pre-installed on Ubuntu Server 24.04 |

---

## Prerequisites

- Ubuntu Server 24.04 with internet access
- User with `sudo` privileges
- `snapd` active — pre-installed, verify before proceeding

```bash
systemctl is-active snapd
snap version
```

Expected: `active` and snap version output.

---

## Phase 1 — System Update

```bash
sudo apt update
sudo apt upgrade -y
```

---

## Phase 2 — Install LXD via Snap

On Ubuntu 24.04, LXD may already be pre-installed. The command below installs it or refreshes to the current stable revision.

```bash
# Install or refresh LXD on the stable snap track
sudo snap install lxd

# Verify installed version
snap list lxd
```

Expected output:

```
Name  Version  Rev    Tracking       Publisher   Notes
lxd   5.21.x   ...    latest/stable  canonical✓  -
```

---

## Phase 3 — User Group Configuration

The `lxd` group must include the current user to run `lxc` commands without `sudo`.

```bash
# Add current user to the lxd group
sudo usermod -aG lxd $USER

# Apply group change to current shell (no logout required)
newgrp lxd

# Confirm group membership
id
```

Expected: `lxd` appears in the groups list.

---

## Phase 4 — Initialize LXD

`lxd init` configures the daemon — storage backend, networking bridge, and access settings.

```bash
sudo lxd init
```

Recommended responses for a standalone homelab VM:

```
Would you like to use LXD clustering? (yes/no) [default=no]: no
Do you want to configure a new storage pool? (yes/no) [default=yes]: yes
Name of the new storage pool [default=default]: default
Name of the storage backend to use [default=dir]: dir
Would you like to connect to a MAAS server? (yes/no) [default=no]: no
Would you like to create a new local network bridge? (yes/no) [default=yes]: yes
What should the new bridge be called? [default=lxdbr0]: lxdbr0
What IPv4 address should be used? [default=auto]: auto
What IPv6 address should be used? [default=auto]: auto
Would you like the LXD server to be available over the network? (yes/no) [default=no]: no
Would you like stale cached images to be updated automatically? [default=yes]: yes
Would you like a YAML "lxd init" preseed to be printed? [default=no]: no
```

> **Storage backend — `dir` vs `zfs`:**
> `dir` stores container filesystems in plain directories. Simple, no extra setup, reliable for lab use.
> `zfs` gives better performance, native copy-on-write snapshots, and faster container clones — but requires ZFS kernel modules and pool configuration. Recommended for heavier workloads or production.

> **arm64 image pulling:**
> All LXD image operations are architecture-aware. When you run `lxc launch ubuntu:24.04`, LXD automatically pulls the `arm64` variant based on the host architecture — no flags needed.

---

## Phase 5 — Validation

### 5.1 Verify LXD Version

```bash
lxd version
```

Expected: `5.21.x` or higher.

### 5.2 Confirm Architecture

```bash
lxc info | grep -i arch
```

Expected: `aarch64` or `arm64`.

### 5.3 List Profiles, Networks, and Storage

```bash
lxc profile list
lxc network list
lxc storage list
```

All three should return at least one default entry each.

### 5.4 Launch a Test Container

```bash
# LXD automatically pulls the arm64 image variant
lxc launch ubuntu:22.04 lab-test

# Verify the container is running
lxc list
```

Expected: `lab-test` in `RUNNING` state with an assigned IPv4 address.

### 5.5 Access the Container

```bash
lxc exec lab-test -- bash

# Verify architecture inside the container
uname -m
cat /etc/os-release
exit
```

Expected: `uname -m` returns `aarch64`.

### 5.6 Stop and Delete the Test Container

```bash
lxc stop lab-test
lxc delete lab-test

# Confirm removal
lxc list
```

---

## Troubleshooting

| Symptom | Root Cause | Resolution |
|---------|-----------|------------|
| `E: Package 'lxd' has no installation candidate` | APT package removed in Ubuntu 22.04+ | Use `sudo snap install lxd` |
| `lxc: command not found` after snap install | `/snap/bin` not in `$PATH` | `export PATH=$PATH:/snap/bin` or re-login |
| `Error: unix.Connect failed` | LXD daemon not running | `sudo snap start lxd` |
| `permission denied` on `lxc` commands | User not in `lxd` group | `sudo usermod -aG lxd $USER && newgrp lxd` |
| Container image pull fails | No arm64 variant for chosen image | Use `ubuntu:22.04` or `ubuntu:24.04` (confirmed arm64 support) |
| Container stuck in `STARTING` | Missing kernel modules | `sudo modprobe br_netfilter && sudo modprobe overlay` |

---

## Environment Considerations

### Lab / Homelab ✅ Recommended

snap is the correct and only supported install method for LXD on Ubuntu 24.04. snapd is pre-installed, LXD snap is actively maintained by Canonical, and the `arm64` variant is fully supported. No workarounds needed. This is the recommended path for any isolated lab or learning environment.

### Staging ⚠️ Conditional

Snap is acceptable in staging if production also uses snap. If your production environment does not use snap — common in regulated environments due to Canonical's control over automatic updates and the snap store — define explicit snap hold policies (`snap refresh --hold`) to prevent unplanned updates during testing windows. Do not rely on default auto-update behavior in staging without validation.

### Production ⚠️ Evaluate carefully

For production LXD deployments on Ubuntu Server, use a dedicated node with `zfs` or `btrfs` as the storage backend — never `dir`. If your organization prohibits snap due to automatic update policies (valid concern: snap updates can trigger unplanned service restarts), evaluate **[Incus](https://linuxcontainers.org/incus/)** — the community fork of LXD, available as a native APT package on Ubuntu 24.04 without snap dependency. Never use Linux Mint as a production base for LXD.

---

## References

- [LXD Official Documentation](https://documentation.ubuntu.com/lxd/latest/)
- [LXD Installation Guide](https://documentation.ubuntu.com/lxd/latest/installing/)
- [LXD First Steps](https://documentation.ubuntu.com/lxd/latest/tutorial/first_steps/)
- [LXD on Snapcraft](https://snapcraft.io/lxd)
- [Incus — LXD Community Fork](https://linuxcontainers.org/incus/)
