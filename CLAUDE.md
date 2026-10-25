# bozov-lab

Home Kubernetes (k3s) lab on 4 Raspberry Pi 5 nodes. Purpose: learn Linux/Kubernetes hands-on and have a place to deploy personal projects. This file is the persistent memory for the project — read it first in any new session, and update it incrementally (don't regenerate it wholesale) as things change, to keep token usage down.

**Never commit secrets** (private keys, passwords, tokens). Document that access exists and how — never the credential material itself.

## Cluster topology

4x Raspberry Pi 5, ARM Cortex-A76 (aarch64). Own dedicated router — LAN-only, not exposed to the internet. Repo is public on GitHub; LAN IPs/hostnames/username below are intentionally included (user accepted that tradeoff — devices still aren't reachable from outside the LAN).

| Role | SSH alias | Hostname | LAN IP | RAM | Boot disk | Extra disk | OS |
|---|---|---|---|---|---|---|---|
| control-plane | `control` | pi5.local | 192.168.0.132 (eth0) / 192.168.68.106 (wifi) | 8GB | 256GB SD | none | Ubuntu 25.04 (non-LTS, EOL ~Jan 2026) |
| worker | `worker1` | pi5-node-1.local | 192.168.0.72 | 16GB | 64GB SD | 512GB NVMe @ `/mnt/nvme` (unused) | Ubuntu 24.04 LTS |
| worker | `worker2` | pi5-node-2.local | 192.168.0.237 | 16GB | 64GB SD | 512GB NVMe @ `/mnt/nvme` (unused) | Ubuntu 24.04 LTS |
| worker | `worker3` | pi5-node-3.local | 192.168.0.37 | 16GB | 64GB SD | 512GB NVMe @ `/mnt/nvme` (unused) | Ubuntu 24.04 LTS |

## How to connect

Passwordless SSH (ed25519 key, no passphrase) + passwordless sudo for user `stbozov` on all 4 nodes. The private key lives only on the administering Windows machine (`~/.ssh/id_ed25519`) — never in this repo.

`~/.ssh/config` on the administering machine:

```
Host control pi5.local
    HostName pi5.local
    User stbozov
    IdentityFile ~/.ssh/id_ed25519

Host worker1 pi5-node-1.local
    HostName pi5-node-1.local
    User stbozov
    IdentityFile ~/.ssh/id_ed25519

Host worker2 pi5-node-2.local
    HostName pi5-node-2.local
    User stbozov
    IdentityFile ~/.ssh/id_ed25519

Host worker3 pi5-node-3.local
    HostName pi5-node-3.local
    User stbozov
    IdentityFile ~/.ssh/id_ed25519
```

Usage: `ssh control "command"`, `ssh worker1 "command"`, etc. `sudo` needs no password (set up via `/etc/sudoers.d/stbozov-nopasswd` on each node).

## Current state (2026-10-25)

- k3s is **not installed yet** on any node.
- OS packages fully updated via `apt full-upgrade` on all 4 nodes — **all 4 need a reboot** (kernel updates) before proceeding.
- NVMe drives on the 3 workers are mounted but idle — plan is to use them as a replicated Longhorn storage pool (see Architecture below), not as the boot/root disk.
- Longhorn prerequisites checked on workers: `open-iscsi` and `cryptsetup` installed; `iscsid` service installed but not enabled/running; `nfs-common` missing; `dm_crypt` kernel module available but not loaded. Need to fix these before installing Longhorn.

## Architecture (agreed plan)

- **Control plane**: `control` runs only the k3s server component — tainted so no workloads are scheduled there, just cluster brain duties.
- **Workers**: `worker1/2/3` run all actual workloads.
- **Storage**: Longhorn, pooling the 3 workers' 512GB NVMe drives (1536GB raw). Default to **2 replicas per volume** (~768GB usable, survives any 1 worker dying) rather than Longhorn's default of 3 (which would waste 2/3 of raw capacity for only 1 node's worth of usable space). Replica count is configurable per storage class/volume, so critical data can use more replicas and scratch data fewer, later.
- **Networking**: k3s defaults for now (Flannel CNI, Traefik ingress). MetalLB is a likely future addition once something real is running, to get LAN IPs for services instead of NodePorts.
- **Repo layout (planned, not yet created)**: `bootstrap/` for node/k3s setup scripts, `manifests/` for what's deployed on the cluster, `.github/workflows/` for automation that should run independent of the home lab's uptime (CI checks, image builds, etc.) rather than on k3s itself.
- **Future, not now**: considered exposing some services publicly later via the ISP (router port-forward) or a Cloudflare Tunnel — no decision made, revisit when there's an actual service worth exposing.

## Decisions log

- Keeping OS version mismatch (`control` on 25.04, workers on 24.04 LTS) rather than reflashing/upgrading in place — not a blocker for k3s. Revisit before 25.04 EOL (~Jan 2026).
- Repo kept public; user explicitly accepted the LAN-topology disclosure.
- Claude Code push access to this repo confirmed working (2026-10-25) via Git Credential Manager — no token handled by Claude.

## Working agreement

- User is new to Linux and Kubernetes (developer background, different stack) — explain steps in plain language, don't assume prior sysadmin/k8s knowledge.
- Decide tooling (Ansible, GitOps, etc.) only when actually needed — nothing picked yet.
