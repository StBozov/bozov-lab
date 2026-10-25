# bozov-lab

Home Kubernetes (k3s) lab on 4 Raspberry Pi 5 nodes. Purpose: learn Linux/Kubernetes hands-on and have a place to deploy personal projects. This file is the persistent memory for the project — read it first in any new session, and update it incrementally (don't regenerate it wholesale) as things change, to keep token usage down.

**Never commit secrets** (private keys, passwords, tokens). Document that access exists and how — never the credential material itself.

## Cluster topology

4x Raspberry Pi 5, ARM Cortex-A76 (aarch64). Own dedicated router — LAN-only, not exposed to the internet. Repo is public on GitHub; LAN IPs/hostnames/username below are intentionally included (user accepted that tradeoff — devices still aren't reachable from outside the LAN).

| Role | SSH alias | Hostname | LAN IP | MAC (eth0) | RAM | Boot disk | Extra disk | OS |
|---|---|---|---|---|---|---|---|---|
| control-plane | `control` | pi5.local | 192.168.0.133 (DHCP reservation, fixed) | `2c:cf:67:75:ef:d4` | 8GB | 256GB SD | none | Ubuntu 25.04 (non-LTS, EOL ~Jan 2026) |
| worker | `worker1` | pi5-node-1.local | 192.168.0.72 (DHCP reservation, fixed) | `88:a2:9e:04:41:93` | 16GB | 64GB SD | 512GB NVMe @ `/mnt/nvme` (not yet in Longhorn pool) | Ubuntu 24.04 LTS |
| worker | `worker2` | pi5-node-2.local | 192.168.0.237 (DHCP reservation, fixed) | `88:a2:9e:04:42:e9` | 16GB | 64GB SD | 512GB NVMe @ `/mnt/nvme` (not yet in Longhorn pool) | Ubuntu 24.04 LTS |
| worker | `worker3` | pi5-node-3.local | 192.168.0.37 (DHCP reservation, fixed) | `88:a2:9e:23:d6:7a` | 16GB | 64GB SD | 512GB NVMe @ `/mnt/nvme` (not yet in Longhorn pool) | Ubuntu 24.04 LTS |

All 4 IPs are reserved by MAC address on the TP-Link router (Address Reservation page), confirmed stable across reboots — no DHCP drift anymore.

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

- **k3s cluster is live.** `control` (k3s server, tainted `node-role.kubernetes.io/control-plane:NoSchedule`) + `worker1/2/3` (k3s agents) all show `Ready`. k3s version v1.36.5+k3s1. Kubeconfig on `control` at `/etc/rancher/k3s/k3s.yaml` (mode 644).
- Known gotcha hit during install: on `control` (Ubuntu 25.04, kernel 6.14), the Raspberry Pi firmware injects `cgroup_disable=memory` into the boot cmdline by default, which the classic `cgroup_enable=memory` cmdline.txt fix does NOT override on this kernel (it did work on the workers' 6.8 kernel). Turned out to be a red herring for the actual k3s failure — cgroup v2's `/sys/fs/cgroup/cgroup.controllers` already listed `memory` as available regardless. The real fix was just a reboot (needed anyway for the `cgroup_memory=1 cgroup_enable=memory` cmdline.txt addition applied to all 4 nodes) — k3s started fine after that.
- Static IPs done: DHCP reservations set on the router for all 4 nodes (by MAC), verified stable across a full reboot of all 4.
- Reboot-resilience verified twice: rebooting all 4 nodes in parallel, k3s (`k3s`/`k3s-agent` systemd services) comes back on its own with no manual steps — cluster self-heals from something like a power outage.
- NVMe drives on the 3 workers are mounted but not yet in use — next step is installing Longhorn to pool them (see Architecture below).
- Longhorn prerequisites checked on workers: `open-iscsi` and `cryptsetup` installed; `iscsid` service installed but not enabled/running; `nfs-common` missing; `dm_crypt` kernel module available but not loaded. Still need to fix these before installing Longhorn.
- Added `.claude/settings.json` (committed) allowing Bash commands prefixed `ssh control/worker1/worker2/worker3` without classifier prompts — scoped to just these 4 SSH aliases. Reading actual secret files (e.g. the k3s node-token) is still hard-blocked regardless of allow rules; the user retrieves those manually when needed.

## Architecture (agreed plan)

- **Control plane**: `control` runs only the k3s server component — tainted so no workloads are scheduled there, just cluster brain duties.
- **Workers**: `worker1/2/3` run all actual workloads.
- **Storage**: Longhorn, pooling the 3 workers' 512GB NVMe drives (1536GB raw). Default to **2 replicas per volume** (~768GB usable, survives any 1 worker dying) rather than Longhorn's default of 3 (which would waste 2/3 of raw capacity for only 1 node's worth of usable space). Replica count is configurable per storage class/volume, so critical data can use more replicas and scratch data fewer, later.
- **Networking**: k3s defaults for now (Flannel CNI, Traefik ingress). MetalLB is a likely future addition once something real is running, to get LAN IPs for services instead of NodePorts.
- **Repo layout (planned, not yet created)**: `bootstrap/` for node/k3s setup scripts, `manifests/` for what's deployed on the cluster, `.github/workflows/` for automation that should run independent of the home lab's uptime (CI checks, image builds, etc.) rather than on k3s itself.
- **Future, not now**: considered exposing some services publicly later via the ISP (router port-forward) or a Cloudflare Tunnel — no decision made, revisit when there's an actual service worth exposing.

## Storage (Longhorn)

Installed via Helm (release `longhorn`, namespace `longhorn-system`), `--set defaultSettings.defaultDataPath=/mnt/nvme/longhorn --set persistence.defaultClassReplicaCount=2 --set defaultSettings.defaultReplicaCount=2`.

- Confirmed: all Longhorn components scheduled only on `worker1/2/3` (the control-plane taint needed no extra tolerations/config to keep `control` clean).
- Each worker has a disk registered at `/mnt/nvme/longhorn` (on the NVMe, not the SD card).
- Default StorageClass `longhorn` uses `numberOfReplicas: "2"` as agreed.
- Longhorn reserves ~140GB/node (25% default safety margin) before replication — real usable pool capacity is **~500GB**, not the naive raw÷2 estimate. Adjustable later (`defaultSettings.storageMinimalAvailablePercentage` / per-disk `storageReserved`) if more usable space is wanted over safety margin.
- Prerequisites fixed on all 3 workers first: `iscsid` enabled+started, `nfs-common` installed, `dm_crypt` module loaded (persisted via `/etc/modules-load.d/dm_crypt.conf`).
- Longhorn's own UI exists too but Headlamp is the primary cluster UI for now; Longhorn UI not separately exposed.

## Cluster UI

**Headlamp** (the official successor to Kubernetes Dashboard, which was archived 2026-01-21) is installed via Helm, release `my-headlamp` in the `kube-system` namespace, bound to `cluster-admin` via ClusterRoleBinding `my-headlamp-admin`, exposed as a NodePort service.

- URL: `http://192.168.0.133:30519` — static (both the node IP and the NodePort are fixed). It's a NodePort service, so Kubernetes opens port `30519` on **all 4 nodes**, not just `control`: `http://192.168.0.72:30519`, `.237:30519`, `.37:30519` all work too, routing to wherever the pod actually is. Useful fallback if `control` is ever down.
- The pod itself (`my-headlamp` deployment, `kube-system` namespace) runs on whichever worker Kubernetes schedules it to — currently `worker1`, but it'll auto-reschedule to another worker if that one goes down. `control` never runs it (tainted). Check with: `kubectl get pods -n kube-system -l app.kubernetes.io/name=headlamp -o wide`.
- Login: paste a token into the browser, generated with `kubectl create token my-headlamp -n kube-system --duration=8760h` (run on `control` with `KUBECONFIG=/etc/rancher/k3s/k3s.yaml`) — **never commit the token itself to this repo**, it's a cluster-admin credential. No server-side session: the browser remembers it via local storage per-browser/device until the token expires or storage is cleared; a new browser/device needs the token pasted again.
- Troubleshooting starting points: `kubectl get pods -n kube-system -l app.kubernetes.io/name=headlamp -o wide` (is it Running, which node), `kubectl get svc my-headlamp -n kube-system` (confirm NodePort still 30519), `kubectl logs -n kube-system -l app.kubernetes.io/name=headlamp` (pod logs), `helm status my-headlamp -n kube-system` (release health).
- Known minor issue: `metrics-server` readiness probe fails intermittently on `control` (common k3s/kubelet TLS quirk) — not yet fixed, cosmetic for now (just means CPU/memory graphs may be incomplete).

## Decisions log

- Keeping OS version mismatch (`control` on 25.04, workers on 24.04 LTS) rather than reflashing/upgrading in place — not a blocker for k3s. Revisit before 25.04 EOL (~Jan 2026).
- Repo kept public; user explicitly accepted the LAN-topology disclosure.
- Claude Code push access to this repo confirmed working (2026-10-25) via Git Credential Manager — no token handled by Claude.

## Working agreement

- User is new to Linux and Kubernetes (developer background, different stack) — explain steps in plain language, don't assume prior sysadmin/k8s knowledge.
- Decide tooling (Ansible, GitOps, etc.) only when actually needed — nothing picked yet.
