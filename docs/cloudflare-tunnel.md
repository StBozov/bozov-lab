# Skill: Public access via Cloudflare Tunnel

How we'll expose cluster services to the public internet, eventually under a real domain. For the cluster itself, see `docs/cluster.md`.

## Status at a glance

- ✅ **Mechanism validated end-to-end** (2026-11-01) via a Cloudflare Quick Tunnel — no domain, no account needed for this test. A disposable test app + `cloudflared` pod inside the cluster produced a public `*.trycloudflare.com` URL that served the app's page. Confirms the whole chain works: outbound tunnel from inside the cluster → Cloudflare edge → back down to the app.
- ✅ **Domain purchased**: `bozov-lab.com`, via Cloudflare Registrar, 2026-11-01 (WHOIS Privacy, DNSSEC, Email Forwarding included). Already on Cloudflare's own DNS, no nameserver-pointing step needed.
- ✅ **Production tunnel live** (2026-11-01): named tunnel `bozov-lab` created, `cloudflared` deployed properly in-cluster, first route mapped and verified — `https://test.bozov-lab.com` serves the test app, confirmed working from inside the home network. See "Production setup" below for full details and next time's gotchas.
- ⏳ **Still to verify**: opening `test.bozov-lab.com` from *outside* the home network (e.g. phone on mobile data) — the home-network test proves the chain works, but doesn't rule out something resolving locally. Worth a quick check when convenient, not blocking.
- 🧹 `tunnel-test` namespace (the nginx "hello" app) is now **actively serving** `test.bozov-lab.com` — don't delete it until replaced with a real app and its route updated accordingly.

## Why Cloudflare Tunnel (not a router port-forward)

Decided against the traditional approach (forward a port on the router to a node, use Dynamic DNS since the router's public IP isn't static) because:
- The router's public IP is dynamic — would need a DDNS service to keep a domain pointed at it.
- Possible CGNAT from the ISP would make inbound connections impossible regardless of port-forwarding — not yet confirmed either way, and irrelevant if Cloudflare Tunnel is used.
- A port-forward directly exposes the home's public IP to internet scanners; Cloudflare Tunnel does not.
- Cloudflare Tunnel also provides TLS and basic DDoS/bot protection for free, with less ongoing maintenance.

Considered Tailscale Funnel as an alternative with similar outbound-tunnel mechanics — not chosen, but worth knowing about if Tailscale ever gets set up for private remote access anyway.

## How the mechanism works (no open ports, survives a dynamic IP)

Traditional port-forwarding needs the internet to know your current public IP (hence DDNS). Cloudflare Tunnel inverts this:

1. `cloudflared` (running as a pod inside the cluster) opens a persistent **outbound** connection to Cloudflare's network — same as any outbound HTTPS/QUIC traffic, nothing for the router to allow inbound.
2. DNS for the public hostname points at Cloudflare's own fixed edge, **never at the home IP**.
3. A visitor's request hits Cloudflare's edge, which looks up which tunnel is registered for that hostname and relays the request down the already-open outbound connection.
4. `cloudflared` receives it inside the cluster and forwards it to the actual Kubernetes Service.

Because the home IP is never referenced anywhere in this chain, **a changing public IP genuinely doesn't matter** — `cloudflared` just reconnects automatically regardless of what IP it reconnects from. No DDNS, no port-forward, no router config at all.

## What was tested (2026-11-01)

1. Deployed a trivial `nginx` Deployment + ClusterIP Service in namespace `tunnel-test`, serving a static "Hello from bozov-lab!" page.
2. Ran `cloudflared` as a one-off pod in the same namespace: `kubectl run cloudflared-quicktest -n tunnel-test --image=cloudflare/cloudflared:latest --restart=Never -- tunnel --url http://hello.tunnel-test.svc.cluster.local:80`
3. Pulled the generated public URL from its logs (`kubectl logs cloudflared-quicktest -n tunnel-test | grep trycloudflare.com`).
4. User to verify by opening that URL from a connection outside the home LAN (e.g. phone on mobile data) — proves it's reaching over the real internet, not just resolving locally.

Quick Tunnel limits (fine for this test, not for production): URL is random and temporary — dies when the pod stops; 200 in-flight request cap; no persistent/custom hostname; no uptime guarantee. This is Cloudflare's own stated intent for Quick Tunnels — testing/dev only.

## Account

- **Domain**: `bozov-lab.com`, registered via Cloudflare Registrar 2026-11-01.
- **Cloudflare account**: logged in via Google OAuth, `st.bozov@gmail.com`.
- **Dashboard**: [dash.cloudflare.com](https://dash.cloudflare.com/) — two separate areas matter here, don't confuse them:
  - **Domains → bozov-lab.com**: domain/zone-level settings — DNS records (viewable, including the ones the tunnel auto-creates), SSL/TLS, Email routing, WHOIS privacy, registration details.
  - **Networking → Tunnels → bozov-lab**: the tunnel itself — Routes (hostname-to-service mappings, this is where new apps get exposed), replicas, live connection logs, token rotation. Adding/editing a route here is what actually creates/updates the DNS record seen in the Domains area — manage tunnel-routed hostnames from here, not by hand-editing DNS records directly, to avoid drift.

## Production setup (what was actually done, 2026-11-01)

**Tunnel**: named `bozov-lab`, tunnel ID `84122607-e444-442f-97e3-300455a4bb2a`. Created via Cloudflare dashboard (Networking → Tunnels → Create a tunnel → connector type `Cloudflared`).

**`cloudflared` deployment**: runs in-cluster, namespace `cloudflare-tunnel`.
- Tunnel token stored as Kubernetes Secret `cloudflared-tunnel-token` (key `token`) — generated from the dashboard's install-command screen, **never committed to this repo**.
- Deployed as a plain `Deployment` (1 replica, image `cloudflare/cloudflared:latest`, args `tunnel --no-autoupdate run`, token injected via `TUNNEL_TOKEN` env var from the Secret) — not the official Helm chart in the end, a minimal manifest was simpler and sufficient.
- Confirmed healthy: dashboard tunnel page shows status "Healthy", 4 registered edge connections, all connectivity pre-checks passing.

**Routing** (how a hostname gets mapped to an app): since `cloudflared` runs in token mode, routes are managed entirely from the Cloudflare dashboard, not a local config file — no redeploy needed to add/change a route.
1. Dashboard → Networking → Tunnels → click the tunnel → **Routes** tab → **+ Add route**.
2. Choose **"Published application"** (not Private hostname/CIDR/Workers VPC — those are for Zero Trust private access, not public exposure).
3. Fill in: Subdomain (e.g. `test`), Domain (`bozov-lab.com`), Service type `HTTP`, Service URL.
4. **Gotcha**: the Service URL field requires a protocol prefix — `http://hello.tunnel-test.svc.cluster.local:80`, not just the bare hostname:port, or it's rejected as invalid.
5. Save — Cloudflare auto-creates the DNS record. Live within roughly a minute.

First route created: `test.bozov-lab.com` → `http://hello.tunnel-test.svc.cluster.local:80` (the `tunnel-test` app). Verified with `curl -s https://test.bozov-lab.com/` returning `200` and the expected page content.

**To expose a new app later**: just repeat the routing steps above with a new subdomain and that app's internal Service address — the tunnel and `cloudflared` deployment don't need to change, one tunnel serves multiple hostnames.

**Troubleshooting `cloudflared` in-cluster**: `kubectl get pods -n cloudflare-tunnel -o wide` (is it Running, which node), `kubectl logs -n cloudflare-tunnel -l app=cloudflared` (connection status, errors), `kubectl rollout restart deployment cloudflared -n cloudflare-tunnel` (force a fresh reconnect if the tunnel ever shows unhealthy in the dashboard). Remember `control` is tainted, so like everything else it'll only ever schedule onto a worker.
