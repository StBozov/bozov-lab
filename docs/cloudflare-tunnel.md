# Skill: Public access via Cloudflare Tunnel

How cluster services get exposed to the public internet. For the cluster itself, see `docs/cluster.md`.

## Status at a glance

- ✅ **Domain**: `bozov-lab.com`, registered via Cloudflare Registrar (WHOIS Privacy, DNSSEC, Email Forwarding included). On Cloudflare's own DNS.
- ✅ **Tunnel live and resilient**: named tunnel `bozov-lab`, `cloudflared` running 2 replicas in-cluster (anti-affinity across different workers, so one worker dying doesn't take the public path down). Survives a full power-cycle of the cluster automatically, no manual steps.
- ✅ **Two live routes**: `crossly.bozov-lab.com` → Traefik (see skill `app-crossly`), and `test.bozov-lab.com` → a minimal `nginx` placeholder app (namespace `tunnel-test`, Deployment/Service both named `hello`). The test route is kept intentionally as a standing proof-of-life for the tunnel — don't delete it without replacing its route. Follow "Walkthrough" below to add a real app's route alongside them.

## Why Cloudflare Tunnel (not a router port-forward)

Decided against the traditional approach (forward a port on the router to a node, use Dynamic DNS since the router's public IP isn't static) because:
- The router's public IP is dynamic — would need a DDNS service to keep a domain pointed at it.
- Possible CGNAT from the ISP would make inbound connections impossible regardless of port-forwarding.
- A port-forward directly exposes the home's public IP to internet scanners; Cloudflare Tunnel does not.
- Cloudflare Tunnel also provides TLS and basic DDoS/bot protection for free, with less ongoing maintenance.

Tailscale Funnel is a similar-mechanics alternative, not chosen — worth knowing about if Tailscale ever gets set up for private remote access anyway.

## How the mechanism works (no open ports, survives a dynamic IP)

Traditional port-forwarding needs the internet to know your current public IP (hence DDNS). Cloudflare Tunnel inverts this:

1. `cloudflared` (running as pods inside the cluster) opens persistent **outbound** connections to Cloudflare's network — same as any outbound HTTPS/QUIC traffic, nothing for the router to allow inbound.
2. DNS for the public hostname points at Cloudflare's own fixed edge, **never at the home IP**.
3. A visitor's request hits Cloudflare's edge, which looks up which tunnel is registered for that hostname and relays the request down an already-open outbound connection.
4. `cloudflared` receives it inside the cluster and forwards it to the actual Kubernetes Service.

Because the home IP is never referenced anywhere in this chain, **a changing public IP genuinely doesn't matter** — `cloudflared` just reconnects automatically regardless of what IP it reconnects from. No DDNS, no port-forward, no router config at all.

## Account

- **Domain**: `bozov-lab.com`.
- **Cloudflare account**: logged in via Google OAuth, `st.bozov@gmail.com`.
- **Dashboard**: [dash.cloudflare.com](https://dash.cloudflare.com/) — two separate areas matter here, don't confuse them:
  - **Domains → bozov-lab.com**: domain/zone-level settings — DNS records (viewable, including the ones the tunnel auto-creates), SSL/TLS, Email routing, WHOIS privacy, registration details.
  - **Networking → Tunnels → bozov-lab**: the tunnel itself — Routes (hostname-to-service mappings, this is where new apps get exposed), replicas, live connection logs, token rotation. Adding/editing a route here is what actually creates/updates the DNS record seen in the Domains area — manage tunnel-routed hostnames from here, not by hand-editing DNS records directly, to avoid drift.

## Production setup

**Tunnel**: named `bozov-lab`, tunnel ID `84122607-e444-442f-97e3-300455a4bb2a`. Created via Cloudflare dashboard (Networking → Tunnels → Create a tunnel → connector type `Cloudflared`).

**`cloudflared` deployment**: runs in-cluster, namespace `cloudflare-tunnel`.
- Tunnel token stored as Kubernetes Secret `cloudflared-tunnel-token` (key `token`) — generated from the dashboard's install-command screen, **never committed to this repo**.
- Plain `Deployment` (2 replicas, image `cloudflare/cloudflared` pinned by version tag and digest in `apps/cloudflare-tunnel/deployment.yaml`, args `tunnel --no-autoupdate run`, token injected via `TUNNEL_TOKEN` env var from the Secret) — not the official Helm chart, a minimal manifest is simpler and sufficient.
- **2 replicas with pod anti-affinity** (`topologyKey: kubernetes.io/hostname`, preferred not required), spread across different workers — so a single node dying doesn't take the tunnel down with it.

**Routing** (how a hostname gets mapped to an app): since `cloudflared` runs in token mode, routes are managed entirely from the Cloudflare dashboard, not a local config file — no redeploy needed to add/change a route.
1. Dashboard → Networking → Tunnels → click the tunnel → **Routes** tab → **+ Add route**.
2. Choose **"Published application"** (not Private hostname/CIDR/Workers VPC — those are for Zero Trust private access, not public exposure).
3. Fill in: Subdomain, Domain (`bozov-lab.com`), Service type `HTTP`, Service URL.
4. **Gotcha**: the Service URL field requires a protocol prefix — `http://<service>.<namespace>.svc.cluster.local:<port>`, not just the bare hostname:port, or it's rejected as invalid.
5. Save — Cloudflare auto-creates the DNS record. Live within roughly a minute.

## Checking availability

Quick health check for any route:

```
curl -s -o /dev/null -w '%{http_code}\n' https://<hostname>.bozov-lab.com/
```

What the response code means:

| Code | Meaning | Where to look |
|---|---|---|
| `200` | Healthy | Normal operation |
| `502` Bad Gateway | Cloudflare reached `cloudflared` fine, but it couldn't reach the backend app | Usually a DNS or app-availability problem inside the cluster. If `control` is down, CoreDNS (which runs there) goes with it — see `docs/cluster.md` → "Known limitation". Otherwise check the target app's pod/service. |
| `530` | Cloudflare can't reach the tunnel at all | `cloudflared` isn't running anywhere reachable — check the cluster is up and the `cloudflared` deployment is healthy. |

Deeper diagnosis when a route isn't healthy:
1. `kubectl get pods -n cloudflare-tunnel -o wide` — is `cloudflared` itself running?
2. `kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide` — is CoreDNS running? (A `502` on an otherwise-healthy cluster points here first.)
3. `kubectl get pods -n <app-namespace> -o wide` — is the actual app running?
4. `kubectl logs -n cloudflare-tunnel -l app=cloudflared --tail=30` — connection status and errors from `cloudflared`'s own perspective.

## Walkthrough: exposing an app publicly

Deploying the app itself (namespace, quota, network policy, Service and Ingress, storage) is covered by the skill `lab-deploy-app`. Exposing it is adding one route; neither touches `cloudflared`.

### Add the route

1. Cloudflare dashboard → Networking → Tunnels → `bozov-lab` → **Routes** tab → **+ Add route** → **Published application**.
2. Subdomain: the app's name (giving `<app>.bozov-lab.com`). Domain: `bozov-lab.com`.
3. Service type `HTTP`. Service URL: the standard target is the ingress controller, `http://traefik.kube-system.svc.cluster.local:80`, and the app's Ingress matches the hostname. Keep the `http://` prefix. Crossly will be the first app to use this; the existing `test` route predates it and points straight at its Service.
4. Save.

### Verify

`curl -s -o /dev/null -w '%{http_code}\n' https://<app>.bozov-lab.com/` — expect `200`.

**Troubleshooting `cloudflared` in-cluster**: `kubectl get pods -n cloudflare-tunnel -o wide` (is it Running, which node), `kubectl logs -n cloudflare-tunnel -l app=cloudflared` (connection status, errors), `kubectl rollout restart deployment cloudflared -n cloudflare-tunnel` (force a fresh reconnect if the tunnel ever shows unhealthy in the dashboard). Remember `control` is tainted, so like everything else it'll only ever schedule onto a worker.

## Working agreement

**Keep this doc lean**: record durable current-state facts (what's live, how it's configured, how to operate and troubleshoot it) — not a log of what was tested and when. If something needs re-verifying, re-check live state rather than trusting a stale historical note.
