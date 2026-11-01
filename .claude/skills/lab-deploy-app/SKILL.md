---
name: lab-deploy-app
description: Rules and checklist for adding or changing any app on the bozov-lab Kubernetes cluster - namespace, resource quota, hostnames, storage, Flux wiring - so apps do not collide, plus the app registry and resource budget. Use when deploying a new app, changing an app's resources, hostname or storage, or checking whether something would collide with another app.
---

# Deploying an app on the lab

Several apps share this cluster. These rules keep them apart. Before choosing a name, hostname or quota, read [registry.md](registry.md); update it in the same commit that changes an app.

## Rules

| Shared thing | Rule |
|---|---|
| Namespace | One per app, named after it. Everything the app owns lives there |
| Resource names | Prefix with the app name (`crossly-ui`), so names stay unique |
| CPU, memory, pods | A ResourceQuota and a LimitRange in the namespace, sized from the budget in the registry. Never deploy an app without them. A quota makes Kubernetes reject any container that declares no CPU or memory, so the LimitRange supplies defaults and a per-container maximum |
| Storage | PVCs use the default Longhorn class (no `storageClassName`). The namespace quota caps the total (`requests.storage`) |
| Public hostname | `<app>.bozov-lab.com`, one route per hostname, listed in the registry. The route is made in the Cloudflare dashboard and points at the ingress controller, `http://traefik.kube-system.svc.cluster.local:80`; the app's own Ingress matches the hostname (see `docs/cloudflare-tunnel.md`). The older `test` route points straight at its Service and predates this standard |
| Ports | No NodePorts for apps: a ClusterIP Service plus an Ingress. (30219 Longhorn and 30519 Headlamp are taken) |
| Network | Default-deny NetworkPolicy for **both ingress and egress** in the namespace, then allow only what the app needs: the ingress controller (Traefik, in `kube-system`) reaching its pods, **DNS (CoreDNS in `kube-system`, port 53 over UDP and TCP) — without it nothing resolves**, traffic between the app's own pods, and any outside service it must call |
| Secrets | Created with `kubectl` in the app's namespace and never committed. Document names and keys, not values |
| GitOps (Flux, once installed) | One Flux Kustomization per app, with its own source path and its own ServiceAccount limited to the app's namespace. Never one object that applies several apps |
| Files in this repo | An app edits only `apps/<app>/`, its own `app-<app>` skill and its own registry row |

## Adding an app

1. Choose the name. Check the registry for the namespace, hostname and ports.
2. Add the app's row with its intended quota, and check that the budget still fits.
3. Create `apps/<app>/` with `namespace.yaml` (label `app.kubernetes.io/part-of: <app>`), `resourcequota.yaml`, `limitrange.yaml` and `networkpolicy.yaml`, then the app's own manifests or Flux wiring.
4. Show the rendered manifests and a server-side dry run (`kubectl apply --dry-run=server`) before applying anything live.
5. Apply, confirm the pods are Ready, and check the namespace events for warnings. Then prove the network policy really enforces: a pod that is not allowed must fail to reach the app, since a policy that is not enforced looks the same as one that is. Test with pods that have been running for at least 20 seconds, because the policy controller programs a new pod's firewall a moment after it starts, and until then it can reach anything. Include a control in a namespace without policies, so a blocked request means the policy and not a broken network.
6. Add the Cloudflare route, and verify it with `curl -I https://<hostname>/`.
7. Write `.claude/skills/app-<app>/SKILL.md` and add a row for it to the router in `CLAUDE.md`.
8. Commit following the git workflow in `CLAUDE.md`.

## Product knowledge stays with the product

An app's own facts (what it consists of, its images and settings, its delivery plan) belong in the app's repository. The `app-<app>` skill holds only what is specific to this cluster and points to the product's skill instead of copying it.
