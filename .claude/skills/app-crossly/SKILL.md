---
name: app-crossly
description: How Crossly is wired on the bozov-lab cluster - namespace and its fencing, the UI, public hostname crossly.bozov-lab.com and its Cloudflare route and Flux sync from the product repository (all applied), secrets and image automation (planned). Use for any change to Crossly on the lab cluster or to find out what of Crossly is deployed there. The product itself (services, images, settings, delivery plan) is documented in the Crossly repository, not here.
---

# Crossly on the lab

Crossly is an open-source cross-stitch pattern editor. **Product knowledge lives in the Crossly hub repository** (GitHub `textyly/crossly`, local clone `C:\dev\textyly\crossly`): read its skill `.claude/skills/crossly-delivery/SKILL.md` and its manifests in `deploy/k8s/`. This skill covers only what is specific to this cluster. Do not copy product facts here. For rules shared by all apps (quota, naming, checklist), use `lab-deploy-app`.

## Wiring

The namespace, quota, limit range, network policies, the Flux sync, the UI and the Cloudflare route are applied. Image automation is planned. The registry in `lab-deploy-app` is the single place for Crossly's quota; do not repeat it here.

| | |
|---|---|
| Namespace | `crossly`, applied by hand from `apps/crossly/` in this repository (the product manifests do not create it). Pod Security enforces `baseline` and only warns and audits `restricted`: the UI meets `restricted`, but the database images will not |
| Public address | `https://crossly.bozov-lab.com`. No email is set up for this domain, so wherever a contact address is required use an existing mailbox |
| Cloudflare route | `crossly.bozov-lab.com` to the ingress controller, `http://traefik.kube-system.svc.cluster.local:80`; the product's Ingress then matches the hostname |
| Ingress host | The product manifests default to `crossly.localhost`. The lab overrides it to `crossly.bozov-lab.com` with a patch in the Flux Kustomization (`apps/crossly/flux-kustomization.yaml`) |
| Images | `ghcr.io/textyly/crossly-<service>`, public, tagged `main-<short-sha>-<unix-timestamp>` (these exist already) |
| Flux | `GitRepository` `crossly` follows `textyly/crossly` branch `main` every 5 minutes, anonymously (public repository, no credentials). `Kustomization` `crossly` applies `deploy/k8s` as ServiceAccount `flux-apply`, with prune and health wait on, so a merge to `main` goes live within minutes. Image automation (planned) will record new tags in `apps/crossly/` in this repository |
| Secrets | Created with `kubectl` in `crossly`. Names are listed here once they exist (none yet) |

## Where things live

- This repository, `apps/crossly/`: the namespace, quota, limit range, network policies and the Flux wiring.
- The Crossly repository, `deploy/k8s/`: the product's own Deployment, Service and Ingress.

## Lab-side work per slice

The product's delivery slices (S0 to S4) are defined in the Crossly repository. These are the parts that happen on this cluster, and the Crossly repository deliberately knows nothing about them:

| Slice | On the lab |
|---|---|
| S0 | The `crossly` namespace with quota, limit range and network policies (`apps/crossly/`); Flux; image automation; the Cloudflare route to Traefik; docs |
| S1 | A Postgres volume sized within the quota; secrets created with `kubectl`; network policies for the real flows |
| S2 | First check that the official MongoDB image runs on the Pi 5's arm64 CPU (it needs a recent ARM core); then the Mongo volume |
| S3 | Google OAuth credentials and consent screen. No email is set up for the domain, so use an existing mailbox for the support and developer contacts |

The dependency runs one way: this repository may refer to Crossly, but the Crossly repositories must never mention this lab, its domain or its hardware.

## State

The UI runs in `crossly` (2 replicas, delivered by Flux) and is served at `https://crossly.bozov-lab.com`; no database or other workload yet. Progress by slice is tracked in the Crossly repository (`PLAN.md`) and on the project board `textyly/projects/1`, not here.

## Checking

```
ssh control "kubectl -n crossly get pods,svc,ingress,resourcequota"
curl -I https://crossly.bozov-lab.com/
```

The cluster is powered off between sessions; if it does not answer, the state above is unverified (see `docs/cluster.md`).
