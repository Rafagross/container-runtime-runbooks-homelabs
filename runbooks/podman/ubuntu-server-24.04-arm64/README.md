# Podman — Ubuntu Server 24.04 LTS (arm64)

**Author:** Rafael Gross  
**Reviewed:** April 2026  
**Platform:** Ubuntu Server 24.04 LTS (Noble Numbat) · arm64 (aarch64) · VirtualBox VM — Homelab  
**Podman Version:** 4.x (Ubuntu Noble universe)

---

## Overview

This runbook covers the complete Podman workflow on Ubuntu Server 24.04 LTS (arm64): installation, image management, and container lifecycle operations.

On Ubuntu 24.04, Podman 4.x is available directly from the `universe` repository — no external repo or GPG key import required. Podman is daemonless and rootless by default. All images used (`alpine`, `nginx`, `busybox`) have official `arm64` variants on Docker Hub — pulled automatically based on host architecture.

---

## Environment

| Property | Value |
|----------|-------|
| OS | Ubuntu Server 24.04 LTS (Noble Numbat) |
| Architecture | arm64 (aarch64) |
| Deployment | VirtualBox VM — Homelab |
| Podman Version | 4.x (Ubuntu Noble universe) |
| Install Method | `apt` — Ubuntu Noble universe (no external repo) |
| Rootless | Yes — no daemon, no sudo required |

---

## Part 1 — Installation

### Prerequisites

```bash
# Verify universe repo is enabled
grep "universe" /etc/apt/sources.list
# Expected: lines with 'noble ... universe multiverse'

# Verify architecture
uname -m
# Expected: aarch64

sudo -v
```

### Phase 1 — System Update

```bash
sudo apt update
sudo apt upgrade -y
```

### Phase 2 — Install Podman

```bash
# Podman 4.x available directly from Ubuntu Noble universe
# No external repo or GPG key import required
sudo apt install -y podman
```

### Phase 3 — Verify Installation

```bash
podman --version
# Expected: podman version 4.x.x

podman info | grep -i rootless
# Expected: rootless: true

podman info | grep -i graphdriver
# Expected: graphDriverName: overlay

# Test with arm64 image (resolved automatically)
podman container run --rm hello-world
```

### Phase 4 — Post-Install Configuration

```bash
# Verify registry config
cat /etc/containers/registries.conf

# Verify short-name aliases
cat /etc/containers/registries.conf.d/000-shortnames.conf | head -10

# Verify slirp4netns (required for rootless networking)
which slirp4netns
# Expected: /usr/bin/slirp4netns
# If missing: sudo apt install -y slirp4netns

# Verify storage config
cat /etc/containers/storage.conf | grep -E "driver|graphRoot"
```

### Installation Validation

| Check | Command | Expected |
|-------|---------|----------|
| Architecture | `uname -m` | `aarch64` |
| Version | `podman --version` | `podman version 4.x.x` |
| Rootless | `podman info \| grep rootless` | `rootless: true` |
| Networking | `which slirp4netns` | `/usr/bin/slirp4netns` |
| Test container | `podman container run --rm hello-world` | Confirmation output |

---

## Part 2 — Image Operations

### Prerequisites

```bash
podman --version
uname -m
# Expected: aarch64

# Internet access required for registry pulls
```

> Images are stored in `~/.local/share/containers/storage/` — separate from Docker's image store.

### Phase 1 — Search and Pull nginx

```bash
# Search registries (docker.io and quay.io by default)
podman search nginx

# Pull official nginx image (arm64 variant resolved automatically)
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

# Pull official alpine image (arm64 resolved automatically)
podman image pull docker.io/library/alpine

# Confirm pull
podman image ls

# Tag with custom name and version
podman image tag alpine myalps:v2

# Confirm tag — appears as localhost/myalps in REPOSITORY column
podman image ls
```

> Podman prefixes locally tagged images with `localhost/` — expected behavior, not an error.

```bash
# Inspect alpine image
podman image inspect alpine
```

Key fields to verify on arm64:
- `Architecture` — must show `arm64`
- `RepoTags` — lists both `docker.io/library/alpine:latest` and `localhost/myalps:v2`
- `RepoDigests` — content-addressable digests per registry and tag

```bash
# Confirm architecture explicitly
podman image inspect alpine --format '{{.Architecture}}'
# Expected: arm64
```

### Phase 3 — Remove Images

```bash
# Force-remove alpine (untags docker.io/library/alpine:latest)
podman image rm -f alpine

# Confirm — localhost/myalps:v2 remains (layers still referenced)
podman images
```

### Image Operations Validation

| Check | Command | Expected |
|-------|---------|----------|
| nginx pulled (arm64) | `podman image inspect docker.io/library/nginx --format '{{.Architecture}}'` | `arm64` |
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
# Expected: container ID (Podman default)
ps
# Detach WITHOUT stopping: Ctrl + p then Ctrl + q

# Inspect configuration
podman container inspect <id_prefix>
```

### Phase 3 — Run with Custom Hostname and Auto-Remove

```bash
podman container run -h alpine-host -it --rm alpine sh
# Inside container:
hostname
# Expected: alpine-host
exit
```

### Phase 4 — Background Container with Logs

```bash
podman container run -d alpine sh -c 'while [ 1 ]; do echo "hello from container"; sleep 1; done'

podman container logs <id_prefix>

podman container ls

podman container stop <id_prefix>
podman container start <id_prefix>

podman container ls
```

### Phase 5 — Auto-Remove on Exit

```bash
podman container run --rm --name auto_rm alpine ping -c 3 google.com

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
| Architecture | `uname -m` | `aarch64` |
| Container created | `podman container ls -a` | STATUS: Created |
| Container running | `podman container ls` | STATUS: Up |
| Custom hostname | `hostname` inside container | `alpine-host` |
| File copy | `podman container exec host-content cat /usr/share/nginx/html/index.html` | `Welcome to my homelab!` |
| After prune | `podman container ls -aq` | Only running IDs |

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `no image found in manifest list for architecture arm64` | Image lacks arm64 variant | Use confirmed arm64 images: `alpine`, `nginx`, `busybox` |
| `Error: short-name did not resolve` | Unqualified image name | Use full path: `docker.io/library/alpine:latest` |
| `podman container exec -ti` returns OCI runtime error | Container already exited | Verify: `podman container ls` first |
| nginx fails to start | Port already bound (rootless) | Use `-p 8080:80` |
| `slirp4netns` not found | Missing rootless networking dependency | `sudo apt install -y slirp4netns` |
| Architecture shows `amd64` | Wrong image or emulation active | `uname -m` → `aarch64`; re-pull image |
| Prune removed unexpected containers | All non-running states removed | Review `podman container ls -a` first |

---

## Notes

> **Rootless by design:** No daemon process runs as root. Each user manages their own container engine instance.

> **No systemd service:** `systemctl is-active podman` returning `inactive` is correct and expected.

> **arm64 image resolution:** Podman uses the host architecture to select the correct image manifest automatically. No `--platform` flag needed for standard images.

> **Image store is separate from Docker:** Images pulled with Podman are not visible to Docker and vice versa.

> **Rootless networking:** Podman uses `slirp4netns` for rootless network namespaces on arm64. If `ping` inside containers fails: `apt list --installed | grep slirp4netns`

> **Detach without stopping:** `Ctrl + p` then `Ctrl + q` — `Ctrl + C` sends SIGINT to the process inside the container and may terminate it.

---

## References

- [Podman Official Documentation](https://docs.podman.io/)
- [Podman Image CLI Reference](https://docs.podman.io/en/latest/markdown/podman-image.1.html)
- [containers/storage](https://github.com/containers/storage)
