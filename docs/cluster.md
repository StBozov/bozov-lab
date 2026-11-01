# Skill: k3s cluster

Everything about the 4-node Raspberry Pi 5 k3s cluster itself — hardware, access, architecture, storage, UI. For exposing cluster services to the public internet, see `docs/cloudflare-tunnel.md` instead.

## Status at a glance

- ✅ One single k3s cluster, 4 nodes, static IPs. Self-recovers automatically from a reboot, a single node going down, or a full power-off/power-on cycle — no manual steps needed in any case.
- ✅ Longhorn storage pooling the 3 workers' NVMe drives (2 replicas, tolerates any 1 worker dying).
- ✅ Headlamp UI reachable from Windows for visual cluster state.
- ✅ Flux (GitOps) installed; Crossly's UI is the first real app, deployed through it. Also running: `cloudflared` and a minimal placeholder app (see `docs/cloudflare-tunnel.md`).
- ⚠️ **Action needed**: `control`'s OS (Ubuntu 25.04) has been EOL/unsupported since 2026-01-15 — no security patches. Should be reflashed to Ubuntu 24.04 LTS (matching the workers) or upgraded to a current release. Not yet done.
- 💤 Cluster is intentionally powered off between sessions (user doesn't want it running 24/7) — if SSH/kubectl aren't responding, check whether it's just off before assuming something's broken. See "Power management".

## Hardware & network

4x Raspberry Pi 5, ARM Cortex-A76 (aarch64). Own dedicated router — LAN-only, not exposed to the internet. Repo is public on GitHub; LAN IPs/hostnames/MACs below are intentionally included (user explicitly accepted that tradeoff — devices still aren't reachable from outside the LAN).

| Role | SSH alias | Hostname | LAN IP (fixed) | MAC (eth0) | RAM | Boot disk | Extra disk | OS |
|---|---|---|---|---|---|---|---|---|
| control-plane | `control` | pi5.local | 192.168.0.133 | `2c:cf:67:75:ef:d4` | 8GB | 256GB SD | none | Ubuntu 25.04 — **EOL since 2026-01-15, unsupported, no security updates** ⚠️ |
| worker | `worker1` | pi5-node-1.local | 192.168.0.72 | `88:a2:9e:04:41:93` | 16GB | 64GB SD | 512GB NVMe, pooled into Longhorn | Ubuntu 24.04 LTS |
| worker | `worker2` | pi5-node-2.local | 192.168.0.237 | `88:a2:9e:04:42:e9` | 16GB | 64GB SD | 512GB NVMe, pooled into Longhorn | Ubuntu 24.04 LTS |
| worker | `worker3` | pi5-node-3.local | 192.168.0.37 | `88:a2:9e:23:d6:7a` | 16GB | 64GB SD | 512GB NVMe, pooled into Longhorn | Ubuntu 24.04 LTS |

All 4 IPs are DHCP reservations by MAC on the TP-Link router (Address Reservation page) — fixed.

## How to connect

Passwordless SSH (ed25519 key, no passphrase) + passwordless sudo for user `stbozov` on all 4 nodes. The private key lives only on the administering Windows machine (`~/.ssh/id_ed25519`) — never in this repo.

`~/.ssh/config` on the administering machine defines aliases `control`, `worker1`, `worker2`, `worker3` (all pointing at `pi5[-node-N].local`, user `stbozov`, that key). Usage: `ssh control "command"`, etc. `sudo` needs no password (`/etc/sudoers.d/stbozov-nopasswd` on each node).

This Claude Code session has a project-level `.claude/settings.json` allowing `ssh control/worker1/worker2/worker3 *` without permission prompts. Reading actual secret files (e.g. the k3s node-token) is still hard-blocked regardless — the user retrieves those manually when needed.

## Cluster architecture

- **Control plane** (`control`): runs only the k3s server component. Tainted `node-role.kubernetes.io/control-plane:NoSchedule` — no workloads scheduled there, just cluster-brain duties.
- **Workers** (`worker1/2/3`): run all actual workloads. ~48GB RAM total available across them for apps (control's 8GB isn't usable for workloads).
- **Storage**: Longhorn, pooling the 3 workers' NVMe drives at `/mnt/nvme/longhorn` (not the SD cards). `longhorn` is the **sole default StorageClass**. k3s's built-in `local-path` addon is **disabled entirely** via `disable: [local-storage]` in `/etc/rancher/k3s/config.yaml` on `control` — just unsetting its default annotation doesn't stick, since k3s regenerates that addon's manifest from its own bundled defaults on every server restart; disabling the addon outright is the only durable fix. 2 replicas per volume by default — tolerates any 1 worker dying, ~500GB real usable capacity after Longhorn's ~140GB/node reserved safety margin (see "Storage math"). Replica count is configurable per storage class/volume if something needs more or less redundancy.
- **Networking**: k3s defaults (Flannel CNI, Traefik ingress). Services exposed via NodePort so far (simple, no DNS/hosts-file setup needed). MetalLB is a likely future addition once there's a reason to want clean LAN IPs instead of NodePorts.
- **Known limitation**: single control-plane node. If `control` goes down, there's no live orchestrator to reschedule anything — including CoreDNS, which runs on `control` — until it comes back. Workers and their already-running pods are unaffected, but internal DNS resolution (and anything depending on it, like the Cloudflare Tunnel reaching an app) breaks until `control` returns. A second control-plane node would fix this; not done.
- **k3s version**: v1.36.5+k3s1. Kubeconfig on `control` at `/etc/rancher/k3s/k3s.yaml` (mode 644, readable without sudo).

### Storage math (why ~500GB, not more)

3 nodes × 512GB NVMe ≈ 1.37 TiB raw. Longhorn reserves ~140GB/node (25% default safety margin) before any replication — ~422GiB reserved total, leaving ~958GiB raw-writable. Since every volume keeps 2 copies, real safe usable capacity for your own unique data is **(1.37TiB − 422GiB) ÷ 2 ≈ ~490GB**. Going past that doesn't break instantly, but it's the planning budget to stay under — beyond it, Longhorn starts refusing new replica placement on tight nodes and risk increases.

## What's installed

**Longhorn** (storage) — Helm release `longhorn`, namespace `longhorn-system`. UI: `http://192.168.0.133:30219` (NodePort — also reachable via any node's IP on that port). **No login by default** — accepted as-is on a private LAN.

**Headlamp** (cluster UI) — the official successor to Kubernetes Dashboard, which was archived 2026-01-21; Helm release `my-headlamp`, namespace `kube-system`, bound to `cluster-admin`. UI: `http://192.168.0.133:30519` (NodePort, also reachable via any node's IP). **Login required**: paste a token, generate with `kubectl create token my-headlamp -n kube-system --duration=8760h` run on `control`. No server-side session — browser remembers it via local storage per-browser/device until expiry or cleared storage. Never commit the token itself.

Troubleshooting Longhorn or Headlamp: `kubectl get pods -n <namespace> -o wide`, `kubectl get svc -n <namespace>`, `kubectl logs`, `helm status <release> -n <namespace>`.

**Flux** (GitOps, v2.10.0) — namespace `flux-system`, only `source-controller` and `kustomize-controller`. Manifests in `apps/flux-system/`: `flux-install.yaml` is generated (`flux install --export --components=source-controller,kustomize-controller`) and about 155 KB, so **never read it**; `kustomization.yaml` next to it adds `--default-service-account=flux-apply`, so a Kustomization that names no ServiceAccount fails instead of running as cluster-admin. Apply and upgrade with `kubectl apply --server-side` (the CRDs are too large for a plain apply). Per-app wiring is in the skill `lab-deploy-app`.

**`cloudflared`** (namespace `cloudflare-tunnel`) also runs in this cluster, connecting it to the public domain `bozov-lab.com` via a Cloudflare Tunnel — see `docs/cloudflare-tunnel.md` for full details, troubleshooting, and how to expose a new app.

## Deploying apps

Adding or changing an app (namespace, quota, hostnames, storage, network policy, GitOps, and avoiding collisions with other apps) is covered by the skill `lab-deploy-app` and its registry. Use that instead of working from this file. App manifests are delivered by Flux (see "What's installed"); platform manifests under `apps/` (such as `apps/crossly/`, `apps/flux-system/`) are applied by hand with `kubectl apply`.

- `.github/workflows/` is reserved for automation that shouldn't depend on the home lab's uptime/power (CI checks, image builds, scheduled jobs hitting external services) — not yet set up, nothing needs it yet.

## Power management

User doesn't want the lab running 24/7 (it's physically in their home).
- **Shutdown**: fully automatable — `ssh <alias> "sudo poweroff"` on all 4 nodes (in parallel, no interdependency).
- **Power-on**: **not automatable**. Raspberry Pi 5 does not support Wake-on-LAN (hardware limitation, not a config issue). Once powered off, a human must physically restore power.
- To close this gap: the user would need smart plugs (e.g. TP-Link Kasa, Shelly) between the wall outlet and each Pi's power supply (or one covering the strip feeding all 4) with a locally-reachable API — not yet purchased/set up.
- Power-on order when restoring manually: `control` first (~30-60s to boot k3s server), then the 3 workers together. Not strictly required (k3s-agent retries connecting), but avoids unnecessary retry noise.

## Decisions log

- OS version mismatch kept (`control` on 25.04, workers on 24.04 LTS) rather than reflashing/upgrading in place. **25.04 has since reached EOL (2026-01-15)** — real action item now, see "Status at a glance".
- Repo kept public; user explicitly accepted the LAN-topology disclosure.
- Claude Code has direct push access to this repo via Git Credential Manager (no token ever handled by Claude).

## Working agreement

- User is a developer, new to Linux and Kubernetes specifically — explain steps in plain language, don't assume prior sysadmin/k8s knowledge. Conceptual "why does this work" questions are common and worth answering thoroughly.
- Verify claims against live cluster state before reporting status — don't assume event history in a UI reflects current state; check and distinguish historical/transient from live issues.
- Prefer parallel operations over sequential when nodes are independent — sequential only matters once workloads/quorum are actually running.
- **Keep this doc lean**: record durable current-state facts (what's true now, how things are configured, how to operate them) — not a log of what was tested and when. If something needs re-verifying, re-check live state rather than trusting a stale historical note.
- Decide additional tooling (Ansible, GitOps, MetalLB, basic auth for Longhorn UI, etc.) only when an actual need arises.
