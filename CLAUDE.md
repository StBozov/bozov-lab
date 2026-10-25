# bozov-lab

Home Kubernetes (k3s) lab on 4 Raspberry Pi 5 nodes. Purpose: learn Linux/Kubernetes hands-on and have a place to deploy personal projects. This file is the persistent memory for the project — **read it first in any new session**, before doing anything else. Update it incrementally (don't regenerate wholesale) as things change, to keep token usage down.

**Never commit secrets** (private keys, tokens, passwords). Document that access exists and how — never the credential material itself.

## Status at a glance

One single k3s cluster, 4 nodes, fully working as of 2026-10-25:
- ✅ Cluster up, static IPs, reboot-resilient (verified twice) and node-failure-resilient (verified: powered off a worker, cluster kept serving, Kubernetes evicted/rescheduled its pods automatically)
- ✅ Longhorn storage installed, pooling the 3 workers' NVMe drives, verified end-to-end (wrote data, killed the pod, data survived)
- ✅ Headlamp UI reachable from Windows for visual cluster state
- ⏳ Not yet done: nothing deployed to the cluster yet beyond infra — **ready to start deploying real apps**
- 🐛 Known minor issue: `metrics-server` readiness probe flaky on `control` (cosmetic, CPU/memory graphs may be incomplete)

## Hardware & network

4x Raspberry Pi 5, ARM Cortex-A76 (aarch64). Own dedicated router — LAN-only, not exposed to the internet. Repo is public on GitHub; LAN IPs/hostnames/MACs below are intentionally included (user explicitly accepted that tradeoff — devices still aren't reachable from outside the LAN).

| Role | SSH alias | Hostname | LAN IP (fixed) | MAC (eth0) | RAM | Boot disk | Extra disk | OS |
|---|---|---|---|---|---|---|---|---|
| control-plane | `control` | pi5.local | 192.168.0.133 | `2c:cf:67:75:ef:d4` | 8GB | 256GB SD | none | Ubuntu 25.04 (non-LTS, EOL ~Jan 2027) |
| worker | `worker1` | pi5-node-1.local | 192.168.0.72 | `88:a2:9e:04:41:93` | 16GB | 64GB SD | 512GB NVMe, pooled into Longhorn | Ubuntu 24.04 LTS |
| worker | `worker2` | pi5-node-2.local | 192.168.0.237 | `88:a2:9e:04:42:e9` | 16GB | 64GB SD | 512GB NVMe, pooled into Longhorn | Ubuntu 24.04 LTS |
| worker | `worker3` | pi5-node-3.local | 192.168.0.37 | `88:a2:9e:23:d6:7a` | 16GB | 64GB SD | 512GB NVMe, pooled into Longhorn | Ubuntu 24.04 LTS |

All 4 IPs are DHCP reservations by MAC on the TP-Link router (Address Reservation page) — fixed, verified stable across reboots and node-down events.

## How to connect

Passwordless SSH (ed25519 key, no passphrase) + passwordless sudo for user `stbozov` on all 4 nodes. The private key lives only on the administering Windows machine (`~/.ssh/id_ed25519`) — never in this repo.

`~/.ssh/config` on the administering machine defines aliases `control`, `worker1`, `worker2`, `worker3` (all pointing at `pi5[-node-N].local`, user `stbozov`, that key). Usage: `ssh control "command"`, etc. `sudo` needs no password (`/etc/sudoers.d/stbozov-nopasswd` on each node).

This Claude Code session has a project-level `.claude/settings.json` allowing `ssh control/worker1/worker2/worker3 *` without permission prompts. Reading actual secret files (e.g. the k3s node-token) is still hard-blocked regardless — the user retrieves those manually when needed.

## Cluster architecture

- **Control plane** (`control`): runs only the k3s server component. Tainted `node-role.kubernetes.io/control-plane:NoSchedule` — no workloads scheduled there, just cluster-brain duties. This needed no extra tolerations/config anywhere else; Longhorn, Headlamp, etc. all respected it automatically.
- **Workers** (`worker1/2/3`): run all actual workloads. ~48GB RAM total available across them for apps (control's 8GB isn't usable for workloads).
- **Storage**: Longhorn, pooling the 3 workers' NVMe drives at `/mnt/nvme/longhorn` (not the SD cards). `longhorn` is the **sole default StorageClass** (k3s's built-in `local-path` had its default annotation removed — two defaults is undefined behavior for PVCs without an explicit class). 2 replicas per volume by default — tolerates any 1 worker dying, ~500GB real usable capacity after Longhorn's ~140GB/node reserved safety margin (see "Storage math" below). Replica count is configurable per storage class/volume if something needs more or less redundancy later.
- **Networking**: k3s defaults (Flannel CNI, Traefik ingress). Services exposed via NodePort so far (simple, no DNS/hosts-file setup needed). MetalLB is a likely future addition once there's a reason to want clean LAN IPs instead of NodePorts.
- **k3s version**: v1.36.5+k3s1. Kubeconfig on `control` at `/etc/rancher/k3s/k3s.yaml` (mode 644, readable without sudo).

### Storage math (why ~500GB, not more)

3 nodes × 512GB NVMe ≈ 1.37 TiB raw. Longhorn reserves ~140GB/node (25% default safety margin) before any replication — ~422GiB reserved total, leaving ~958GiB raw-writable. Since every volume keeps 2 copies, real safe usable capacity for your own unique data is **(1.37TiB − 422GiB) ÷ 2 ≈ ~490GB**. Going past that doesn't break instantly, but it's the planning budget to stay under — beyond it, Longhorn starts refusing new replica placement on tight nodes and risk increases.

## What's installed

**Longhorn** (storage) — Helm release `longhorn`, namespace `longhorn-system`. UI: `http://192.168.0.133:30219` (NodePort — also reachable via any node's IP on that port). **No login by default** — accepted as-is on a private LAN.

**Headlamp** (cluster UI) — the official successor to Kubernetes Dashboard, which was archived 2026-01-21; Helm release `my-headlamp`, namespace `kube-system`, bound to `cluster-admin`. UI: `http://192.168.0.133:30519` (NodePort, also reachable via any node's IP). **Login required**: paste a token, generate with `kubectl create token my-headlamp -n kube-system --duration=8760h` run on `control`. No server-side session — browser remembers it via local storage per-browser/device until expiry or cleared storage. Never commit the token itself.

Troubleshooting either: `kubectl get pods -n <namespace> -o wide`, `kubectl get svc -n <namespace>`, `kubectl logs`, `helm status <release> -n <namespace>`.

## Validated so far

- **Reboot resilience** (all 4 nodes, twice): k3s/k3s-agent systemd services come back with zero manual steps — survives something like a power outage.
- **Node-failure resilience**: physically powered off `worker3` — the other 3 nodes kept working, `kubectl`/Headlamp stayed reachable throughout, Kubernetes correctly marked it `NotReady` and (after the ~5min grace period) reschedules its pods onto healthy workers.
- **Storage persistence**: PVC + pod via the `longhorn` StorageClass, wrote data, deleted and recreated the pod — data survived. Namespace cleaned up afterward (`longhorn-test`, no longer exists).

## Deploying apps — conventions going forward

Nothing's deployed yet beyond infra. Plan for when real apps go in:
- **Manifests live in this repo** under `apps/<app-name>/*.yaml` (not yet created — create when the first app is added). Applied manually via `kubectl apply -f` for now; no GitOps (Flux/Argo) yet — revisit if that becomes worth the overhead.
- **One namespace per app** is the default pattern (matches what we did for the Longhorn test).
- Don't specify `storageClassName` unless you deliberately want something other than Longhorn's default resilient storage.
- `.github/workflows/` is reserved for automation that shouldn't depend on the home lab's uptime/power (CI checks, image builds, scheduled jobs hitting external services) — not yet set up, nothing needs it yet.

## Decisions log

- OS version mismatch kept (`control` on 25.04, workers on 24.04 LTS) rather than reflashing/upgrading in place — not a blocker for k3s. Revisit before 25.04 EOL.
- Repo kept public; user explicitly accepted the LAN-topology disclosure.
- Claude Code has direct push access to this repo via Git Credential Manager (no token ever handled by Claude).
- k3s install gotcha (informational, already resolved): on `control`'s 6.14 kernel, Raspberry Pi firmware injects `cgroup_disable=memory` into the boot cmdline; the classic `cgroup_enable=memory` fix didn't override it there (did work on workers' 6.8 kernel) — turned out to be a red herring, cgroup v2's controller was fine regardless. Real fix was just the reboot already needed for other reasons.
- Considered exposing services publicly later via ISP router port-forward or a Cloudflare Tunnel — no decision made, revisit when there's an actual service worth exposing. Leaning Cloudflare Tunnel for avoiding direct home-IP exposure, when that time comes.

## Working agreement

- User is a developer, new to Linux and Kubernetes specifically — explain steps in plain language, don't assume prior sysadmin/k8s knowledge. Conceptual "why does this work" questions are common and worth answering thoroughly (e.g. how StorageClass/CSI registration makes storage available cluster-wide, not per-app).
- Verify claims against live cluster state before reporting status — don't assume event history in the UI reflects current state; check and distinguish historical/transient from live issues.
- Prefer parallel operations over sequential when nodes are independent (e.g. rebooting multiple nodes with no interdependency) — sequential only matters once workloads/quorum are actually running.
- Decide additional tooling (Ansible, GitOps, MetalLB, basic auth for Longhorn UI, etc.) only when an actual need arises — nothing beyond what's listed above is planned yet.
