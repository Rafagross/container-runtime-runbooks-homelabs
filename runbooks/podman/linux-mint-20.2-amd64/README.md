# Podman — Linux Mint 20.2 (Uma) · amd64

**Author:** Rafael Gross  
**Reviewed:** April 2026  
**Platform:** Linux Mint 20.2 (Uma) — Ubuntu 20.04 Focal base · amd64 · Physical — OptiPlex Homelab  
**Podman Version:** 3.4.x (podman-static binary)

---

## Overview

This runbook covers the complete Podman workflow on Linux Mint 20.2: installation, image management, and container lifecycle operations.

**Important install context:** `podman` is not available in Ubuntu 20.04 Focal `universe` (only from 20.10+). The Kubic OBS repository is deprecated and returns 404/GPG errors on current systems. The correct install method on Focal is the statically linked binary from `mgoltzsche/podman-static`, which bundles all required dependencies.

Podman is daemonless and rootless by default — no `sudo` required for container and image operations. There is no background service to manage.

---

## Environment

| Property | Value |
|----------|-------|
| OS | Linux Mint 20.2 (Uma) — Ubuntu 20.04 Focal base |
| Architecture | amd64 |
| Deployment | Physical — OptiPlex Homelab |
| Podman Version | 3.4.x (podman-static) |
| Install Method | Binary tarball — `mgoltzsche/podman-static` GitHub releases |
| Rootless | Yes — no daemon, no sudo required |

---

## Part 1 — Installation

### Prerequisites

```bash
# Verify runc is available (used as fallback runtime)
runc --version

# Verify no stale Kubic repo files remain
ls /etc/apt/sources.list.d/ | grep kubic
# Expected: no output

sudo -v
```

### Phase 1 — System Update

```bash
sudo apt update
sudo apt upgrade -y
```

### Phase 2 — Install Podman

```bash
# Step 1 — Install uidmap (required for rootless user namespace mapping)
sudo apt install -y uidmap

# Step 2 — Verify subuid/subgid mappings exist for your user
grep $USER /etc/subuid
grep $USER /etc/subgid
# Expected: <username>:100000:65536
# If missing:
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER

# Step 3 — Download latest podman-static amd64 release
# Includes: runc, crun, conmon, fuse-overlayfs, netavark, pasta, aardvark-dns
curl -fsSL -o podman-linux-amd64.tar.gz \
  https://github.com/mgoltzsche/podman-static/releases/latest/download/podman-linux-amd64.tar.gz

# Step 4 — Extract and install (binaries → /usr/local/bin, config → /etc/containers)
tar -xzf podman-linux-amd64.tar.gz
sudo cp -r podman-linux-amd64/usr podman-linux-amd64/etc /

# Step 5 — Clean up
rm -rf podman-linux-amd64 podman-linux-amd64.tar.gz
```

### Phase 3 — Verify Installation

```bash
podman --version
# Expected: podman version 3.4.x

podman info | grep -i rootless
# Expected: rootless: true

podman info | grep -i graphdriver
# Expected: graphDriverName: overlay (or fuse-overlayfs)

# The cgroups-v1 warning is expected on Focal — informational only, not an error
# WARN[0000] Using cgroups-v1 which is deprecated in favor of cgroups-v2...
PODMAN_IGNORE_CGROUPSV1_WARNING=1 podman container run --rm hello-world
```

### Phase 4 — Post-Install Configuration

```bash
# Verify registry config
cat /etc/containers/registries.conf

# Verify short-name aliases (resolves 'alpine', 'nginx', etc.)
cat /etc/containers/registries.conf.d/000-shortnames.conf | head -10

# Verify storage config
cat /etc/containers/storage.conf | grep -E "driver|graphRoot"
```

### Installation Validation

| Check | Command | Expected |
|-------|---------|----------|
| Version | `podman --version` | `podman version 3.4.x` |
| Rootless | `podman info \| grep rootless` | `rootless: true` |
| No daemon | `systemctl is-active podman` | `inactive` — correct, no daemon |
| Test container | `PODMAN_IGNORE_CGROUPSV1_WARNING=1 podman container run --rm hello-world` | Confirmation output |

---

## Part 2 — Image Operations

### Prerequisites

```bash
podman --version
# Internet access required for registry pulls
```

> Images are stored in `~/.local/share/containers/storage/` — separate from Docker's image store.

### Phase 1 — Search and Pull nginx

```bash
# Search registries (docker.io and quay.io by default)
podman search nginx

# Pull official nginx image
podman image pull docker.io/library/nginx

# List cached images
podman image ls

# List with digests
podman image ls --digests
```

### Phase 2 — Pull, Tag, and Inspect Alpine

```bash
# Search for alpine
podman search alpine

# Pull official alpine image
podman image pull docker.io/library/alpine

# Confirm pull
podman image ls

# Tag with custom name and version
podman image tag alpine myalps:v2

# Confirm tag — appears as localhost/myalps in REPOSITORY column
podman image ls
```

> Podman prefixes locally tagged images with `localhost/` — this is expected behavior, not an error.

```bash
# Inspect alpine image
podman image inspect alpine
```

Key differences vs Docker inspect output:
- `Id` field has no `sha256:` prefix in Podman
- `RepoDigests` lists entries from both `docker.io/library/alpine` and `localhost/myalps`
- `Digest` appears at top level

### Phase 3 — Remove Images

```bash
# Force-remove alpine (untags docker.io/library/alpine:latest)
podman image rm -f alpine

# Confirm — localhost/myalps:v2 remains (layers still referenced by that tag)
podman images
```

### Image Operations Validation

| Check | Command | Expected |
|-------|---------|----------|
| nginx pulled | `podman image ls` | `docker.io/library/nginx latest` listed |
| Tag created | `podman image ls` | `localhost/myalps v2` with same IMAGE ID as alpine |
| Both tags in inspect | `podman image inspect alpine` | Both tags in `RepoTags` |
| alpine removed, tag remains | `podman images` | Only `localhost/myalps:v2` shown |

---

## Part 3 — Container Operations

### Phase 1 — Create and Start

```bash
# Create container without starting it
podman container create -it alpine sh

# Confirm created but not running
podman container ls
# Expected: empty

podman container ls -a
# Expected: STATUS: Created

# Start using ID prefix
podman container start <id_prefix>

podman container ls
# Expected: STATUS: Up
```

### Phase 2 — Attach and Inspect

```bash
# Attach to stdin/stdout
podman container attach <id_prefix>
# Inside container:
hostname
# Expected: container ID (Podman default — differs from Docker)
ps
# Detach WITHOUT stopping: Ctrl + p then Ctrl + q

# Inspect configuration
podman container inspect <id_prefix>
```

### Phase 3 — Run with Custom Hostname and Auto-Remove

```bash
# Custom hostname; auto-removed on exit
podman container run -h alpine-host -it --rm alpine sh
# Inside container:
hostname
# Expected: alpine-host
exit
# Container removed automatically
```

### Phase 4 — Background Container with Logs

```bash
# Run detached looping container
podman container run -d alpine sh -c 'while [ 1 ]; do echo "hello from container"; sleep 1; done'

# View logs
podman container logs <id_prefix>

podman container ls

# Stop and restart
podman container stop <id_prefix>
podman container start <id_prefix>

podman container ls
```

### Phase 5 — Auto-Remove on Exit

```bash
podman container run --rm --name auto_rm alpine ping -c 3 google.com

# Confirm removed
podman container ls -a | grep auto_rm
# Expected: no output
```

### Phase 6 — Force Remove Running Container

```bash
podman container rm -f <id_prefix>

podman container ls
```

### Phase 7 — Exec and File Copy

```bash
# Create file on host
echo "Welcome to my homelab!" > ~/host-file

# Run nginx in background
podman container run -d --name host-content nginx

# Copy file into container
podman container cp ~/host-file host-content:/usr/share/nginx/html/index.html

# Validate non-interactively
podman container exec host-content cat /usr/share/nginx/html/index.html
# Expected: Welcome to my homelab!

# Validate interactively
podman container exec -ti host-content sh
cat /usr/share/nginx/html/index.html
exit
```

### Phase 8 — Prune Non-Running Containers

```bash
podman container ls -aq

podman container prune
# Confirm: y

podman container ls -aq
# Expected: only running container IDs remain
```

> `podman container prune` removes ALL non-running containers including `Created` state. Always review `podman container ls -a` before running in shared environments.

### Container Operations Validation

| Check | Command | Expected |
|-------|---------|----------|
| Container created | `podman container ls -a` | STATUS: Created |
| Container running | `podman container ls` | STATUS: Up |
| Custom hostname | `hostname` inside container | `alpine-host` |
| Logs | `podman container logs <id>` | Repeated output lines |
| File copy | `podman container exec host-content cat /usr/share/nginx/html/index.html` | `Welcome to my homelab!` |
| After prune | `podman container ls -aq` | Only running IDs |

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `E: Unable to locate package podman` | Not in Focal universe | Use podman-static tarball (see Part 1) |
| `exec: "newuidmap": not found` | `uidmap` not installed | `sudo apt install -y uidmap` then configure `/etc/subuid` and `/etc/subgid` |
| `WARN: Using cgroups-v1` | Focal uses cgroups v1 — informational only | `export PODMAN_IGNORE_CGROUPSV1_WARNING=1` to suppress |
| `Error: short-name did not resolve` | Unqualified image name | Use full path: `docker.io/library/alpine:latest` |
| Short ID ambiguity | Two containers share prefix | Use more characters of the ID |
| `--rm` container can't restart | Auto-deleted on exit by design | Omit `--rm` if restart is needed |
| `prune` removed unexpected containers | Removes all non-running states | Review `podman container ls -a` first |

### Clean Reset (stale Kubic artifacts)

```bash
sudo rm -f /etc/apt/sources.list.d/*kubic*
sudo rm -f /etc/apt/keyrings/kubic.gpg
sudo apt update
# Then reinstall via podman-static (see Part 1, Phase 2)
```

---

## Notes

> **Rootless by design:** No daemon process runs as root. Each user manages their own container engine instance. Safer than Docker's daemon model for multi-user systems.

> **No systemd service:** `systemctl is-active podman` returning `inactive` is correct and expected. There is nothing to start or enable.

> **Image store is separate from Docker:** Images pulled with Podman are not visible to Docker and vice versa. They use completely separate storage paths.

> **cgroups-v1 on Focal:** Linux Mint 20.2 (Focal base) uses cgroups v1 by default. Podman v5 deprecated cgroups-v1 support. The warning is informational — all operations work correctly. Set `PODMAN_IGNORE_CGROUPSV1_WARNING=1` to suppress it permanently in your shell profile.

> **Detach without stopping:** `Ctrl + p` then `Ctrl + q` — this is the correct sequence. `Ctrl + C` sends SIGINT to the process inside the container and may terminate it.

---

## References

- [Podman Official Documentation](https://docs.podman.io/)
- [podman-static — Static Podman Binaries](https://github.com/mgoltzsche/podman-static)
- [Podman Image CLI Reference](https://docs.podman.io/en/latest/markdown/podman-image.1.html)
- [containers/storage](https://github.com/containers/storage)
