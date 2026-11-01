# bozov-lab

Home Kubernetes (k3s) lab on 4 Raspberry Pi 5 nodes. Purpose: learn Linux/Kubernetes hands-on and have a place to deploy personal projects. This file is the persistent memory for the project — **read it first in any new session**, before doing anything else.

**Never commit secrets** (private keys, tokens, passwords). Document that access exists and how — never the credential material itself.

## Skills (topic docs)

Detailed, topic-focused docs live under `docs/`. Read the relevant one before working in that area — keeps this root file small and keeps token usage down. Update docs incrementally (don't regenerate wholesale) as things change.

| Skill | File | Covers |
|---|---|---|
| k3s cluster | [`docs/cluster.md`](docs/cluster.md) | Hardware, network, SSH access, cluster architecture, storage (Longhorn), cluster UI (Headlamp), power management, decisions log |
| Public access via Cloudflare Tunnel | [`docs/cloudflare-tunnel.md`](docs/cloudflare-tunnel.md) | Exposing cluster services to the internet without a port-forward — mechanism, current routes, how to add a new one |

More skill docs get added here as the lab grows (e.g. a future "deploying apps" or "GitOps" doc, if those become real).

**Git workflow**: this is a personal repository (`StBozov/bozov-lab` on GitHub, default branch `main`), so commit and push **straight to `main`** — no branches, no pull requests. Never commit or push without first showing the user what will go in and getting an explicit "yes" for that commit; this applies to manifests and scripts as well as docs. Keep the existing git identity (`stbozov`).

**Review-before-push workflow**: skill docs must be kept up to date on every major change. Before committing/pushing an update to any skill doc (this file or anything under `docs/`), show the user the proposed change and wait for explicit approval — only commit and push after they say yes. Minor in-progress edits don't need to round-trip for approval each time, but the final version going to git does.

**Keep skill docs lean**: every skill doc should stay as small as token budget allows *without* dropping anything that's actually useful — the test is "is this a durable fact about current state/config/how-to," not "is this interesting." Record what's true now (e.g. "`test.bozov-lab.com` is live and routes to X") and how things are configured/operated — never a log of what was tested, when, or how it was verified (e.g. not "we rebooted the cluster on 2026-11-01 and it recovered in 16 seconds"). If a doc accumulates that kind of history, strip it on the next major edit rather than letting it grow. If something needs re-confirming, re-check live state instead of trusting a stale note.

## Always-relevant reminders

- 💤 The cluster is intentionally powered off between sessions (home lab, not meant to run 24/7) — if SSH/kubectl aren't responding, check `docs/cluster.md` → Power management before assuming something's broken.
- ⚠️ `control`'s OS (Ubuntu 25.04) has been EOL/unsupported since 2026-01-15 — outstanding action item, see `docs/cluster.md`.
- User is a developer, new to Linux and Kubernetes specifically — explain steps in plain language, don't assume prior sysadmin/k8s knowledge.
