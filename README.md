# Container Runtime Runbooks — Homelabs

> Practical, production-minded runbooks for installing and configuring container runtimes on real hardware.

**Author:** Rafael Gross  
**Role:** Cloud Operations Engineer · AWS · GCP · Kubernetes  
**Last Updated:** April 2026

---

## About

This repository contains step-by-step runbooks for deploying container runtimes in homelab environments. Every procedure here has been executed, debugged, and validated on physical hardware — not documentation copied from upstream sources.

Where official documentation was outdated, incomplete, or simply wrong for specific OS configurations, the correct working procedure is documented with a full root cause explanation.

---

## Runbooks

### Container Runtimes

| Runbook | Runtime | Platform | Architecture |
|---------|---------|----------|--------------|
| [Install CRI-O](runbooks/cri-o/linux-mint-20.2-amd64/) | CRI-O v1.32 | Linux Mint 20.2 (Uma) | amd64 |
| [Install CRI-O](runbooks/cri-o/ubuntu-server-24.04-arm64/) | CRI-O v1.32 | Ubuntu Server 24.04 LTS | arm64 |

### CLI Tools

| Runbook | Tool | Platform | Architecture |
|---------|------|----------|--------------|
| [Install crictl](runbooks/crictl/linux-mint-20.2-amd64/) | crictl v1.32 | Linux Mint 20.2 (Uma) | amd64 |
| [Install crictl](runbooks/crictl/ubuntu-server-24.04-arm64/) | crictl v1.32 | Ubuntu Server 24.04 LTS | arm64 |

### Container Managers

| Runbook | Tool | Platform | Architecture |
|---------|------|----------|--------------|
| [Install LXD](runbooks/lxd/linux-mint-20.2-amd64/) | LXD 5.21 LTS | Linux Mint 20.2 (Uma) | amd64 |
| [Install LXD](runbooks/lxd/ubuntu-server-24.04-arm64/) | LXD 5.21 LTS | Ubuntu Server 24.04 LTS | arm64 |

### Podman

| Runbook | Covers | Platform | Architecture |
|---------|--------|----------|--------------|
| [Podman — Linux Mint 20.2](runbooks/podman/linux-mint-20.2-amd64/) | Install · Image Ops · Container Ops | Linux Mint 20.2 (Uma) | amd64 |
| [Podman — Ubuntu Server 24.04](runbooks/podman/ubuntu-server-24.04-arm64/) | Install · Image Ops · Container Ops | Ubuntu Server 24.04 LTS | arm64 |

> Additional runbooks will be added as they are completed and validated.

---

## Repository Structure

```
container-runtime-runbooks-homelabs/
└── runbooks/
    ├── cri-o/
    │   ├── linux-mint-20.2-amd64/
    │   └── ubuntu-server-24.04-arm64/
    ├── crictl/
    │   ├── linux-mint-20.2-amd64/
    │   └── ubuntu-server-24.04-arm64/
    ├── lxd/
    │   ├── linux-mint-20.2-amd64/
    │   └── ubuntu-server-24.04-arm64/
    └── podman/
        ├── linux-mint-20.2-amd64/
        │   └── README.md  ← install + image ops + container ops
        └── ubuntu-server-24.04-arm64/
            └── README.md  ← install + image ops + container ops
```

---

## Environments Covered

| OS | Version | Architecture | Notes |
|----|---------|--------------|-------|
| Linux Mint | 20.2 Uma | amd64 | Ubuntu 20.04 (Focal) base |
| Ubuntu Server | 24.04 LTS Noble | arm64 | VirtualBox VM — Homelab |

---

## Standards Applied

- All commands tested on the exact OS and kernel versions listed per runbook
- Root cause documented for every known failure mode
- Troubleshooting tables included in every runbook
- Environment guidance (Lab / Staging / Production) provided where relevant
- No vendor-specific tooling assumptions — plain `bash`, `apt`, `systemctl`, `snap`

---

## License

MIT — free to use, adapt, and build on.
