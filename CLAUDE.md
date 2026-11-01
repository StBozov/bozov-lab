# bozov-lab

Home Kubernetes (k3s) lab on 4 Raspberry Pi 5 nodes. Purpose: learn Linux/Kubernetes hands-on and have a place to deploy personal projects. This file is the persistent memory for the project — **read it first in any new session**, before doing anything else.

**Never commit secrets** (private keys, tokens, passwords). Document that access exists and how — never the credential material itself.

## Load only what the task needs

Detailed knowledge is split into skills (`.claude/skills/<name>/`, listed by the tool with a one-line description) and topic docs (`docs/`). Read the one that matches the task, not all of them. Naming: `lab-*` is shared platform knowledge, `app-*` is one app's wiring on this cluster. Add a row here when you add one.

| Task | Use |
|---|---|
| Add or change any app, check for collisions, resource budget, app registry | skill `lab-deploy-app` (and its `registry.md`) |
| Anything about Crossly on this cluster | skill `app-crossly` |
| Hardware, network, SSH, storage (Longhorn), Headlamp, power management | [`docs/cluster.md`](docs/cluster.md) |
| Public access, Cloudflare Tunnel, adding a route | [`docs/cloudflare-tunnel.md`](docs/cloudflare-tunnel.md) |
| Flux install or upgrade | `docs/cluster.md` → What's installed. Never read `apps/flux-system/flux-install.yaml` (generated, 155 KB) |

**Git workflow**: this is a personal repository (`StBozov/bozov-lab`, default branch `main`). Commit and push **straight to `main`** — no branches, no pull requests. Never commit or push without first showing the user what will go in (docs, manifests and scripts alike) and getting an explicit "yes" for that commit. Keep docs and skills up to date on every major change; minor in-progress edits need no round-trip, but the final version going to git does. Keep the existing git identity (`stbozov`).

**Keep skills and docs lean**: every one should stay as small as token budget allows *without* dropping anything that's actually useful — the test is "is this a durable fact about current state/config/how-to," not "is this interesting." Record what's true now (e.g. "`test.bozov-lab.com` is live and routes to X") and how things are configured/operated — never a log of what was tested, when, or how it was verified. If one accumulates that kind of history, strip it on the next major edit. If something needs re-confirming, re-check live state instead of trusting a stale note.

**State**: what the docs and the registry say about the cluster is what was last confirmed, not a live reading. The live cluster is the source of truth: check it yourself with `ssh control "kubectl ..."` and `curl` before relying on or reporting it. If the lab is unreachable, nothing can be confirmed: report the state as "as last recorded, unverified", do not guess or act as if it were current, and do not treat the silence as a fault, since the lab is off between sessions. Update the "last confirmed" line in the registry whenever you check.

**Comments**: avoid them, in manifests, scripts and any code. Make the file explain itself with clear names, small steps and one concern per object. Add a comment only where the logic is genuinely hard to reason about (a non-obvious constraint, a workaround, a surprising reason), and say *why*, never *what*. Put the rationale for a decision in the relevant skill or doc, not in the file.

## Always-relevant reminders

- 💤 The cluster is intentionally powered off between sessions (home lab, not meant to run 24/7) — if SSH/kubectl aren't responding, check `docs/cluster.md` → Power management before assuming something's broken.
- ⚠️ `control`'s OS (Ubuntu 25.04) has been EOL/unsupported since 2026-01-15 — outstanding action item, see `docs/cluster.md`.
- User is a developer, new to Linux and Kubernetes specifically — explain steps in plain language, don't assume prior sysadmin/k8s knowledge.
