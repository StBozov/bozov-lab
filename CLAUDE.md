# bozov-lab

Home Kubernetes (k3s) lab on 4 Raspberry Pi 5 nodes. Purpose: learn Linux/Kubernetes hands-on and have a place to deploy personal projects. This file is the persistent memory for the project — **read it first in any new session**, before doing anything else.

**Never commit secrets** (private keys, tokens, passwords). Document that access exists and how — never the credential material itself.

## Skills (topic docs)

Detailed, topic-focused docs live under `docs/`. Read the relevant one before working in that area — keeps this root file small and keeps token usage down. Update docs incrementally (don't regenerate wholesale) as things change.

| Skill | File | Covers |
|---|---|---|
| k3s cluster | [`docs/cluster.md`](docs/cluster.md) | Hardware, network, SSH access, cluster architecture, storage (Longhorn), cluster UI (Headlamp), power management, decisions log |
| Public access via Cloudflare Tunnel | [`docs/cloudflare-tunnel.md`](docs/cloudflare-tunnel.md) | Exposing cluster services to the internet without a port-forward — mechanism, what's been tested, plan for the real setup |

More skill docs get added here as the lab grows (e.g. a future "deploying apps" or "GitOps" doc, if those become real).

## Always-relevant reminders

- 💤 The cluster is intentionally powered off between sessions (home lab, not meant to run 24/7) — if SSH/kubectl aren't responding, check `docs/cluster.md` → Power management before assuming something's broken.
- ⚠️ `control`'s OS (Ubuntu 25.04) has been EOL/unsupported since 2026-01-15 — outstanding action item, see `docs/cluster.md`.
- User is a developer, new to Linux and Kubernetes specifically — explain steps in plain language, don't assume prior sysadmin/k8s knowledge.
