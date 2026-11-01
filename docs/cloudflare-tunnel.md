# Skill: Public access via Cloudflare Tunnel

How we'll expose cluster services to the public internet, eventually under a real domain. For the cluster itself, see `docs/cluster.md`.

## Status at a glance

- ✅ **Mechanism validated end-to-end** (2026-11-01) via a Cloudflare Quick Tunnel — no domain, no account needed for this test. A disposable test app + `cloudflared` pod inside the cluster produced a public `*.trycloudflare.com` URL that served the app's page. Confirms the whole chain works: outbound tunnel from inside the cluster → Cloudflare edge → back down to the app.
- ✅ **Domain purchased**: `bozov-lab.com`, via Cloudflare Registrar, 2026-11-01 (WHOIS Privacy, DNSSEC, Email Forwarding included). Already on Cloudflare's own DNS, no nameserver-pointing step needed.
- ⏳ **Not yet done**: no named (persistent) tunnel created, no production `cloudflared` deployment, no hostname mapped. Nothing is currently exposed to the internet under this domain.
- 🧹 The test namespace (`tunnel-test`: a trivial nginx "hello" app + a one-off `cloudflared-quicktest` pod) may still exist — clean up with `kubectl delete namespace tunnel-test` once no longer needed for reference.

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
- **Dashboard**: [dash.cloudflare.com](https://dash.cloudflare.com/) — domain registration details and DNS records are managed from the domain's menu there.

## Plan for the real (production) setup

1. ✅ **Buy a domain** — `bozov-lab.com`, done.
2. ✅ **Cloudflare account set up**, domain is already a zone there (automatic, since it was bought through Cloudflare directly).
3. **Create a named Tunnel** in the Cloudflare dashboard (Zero Trust → Networks → Tunnels) — generates a tunnel ID + credentials token (persistent, unlike Quick Tunnels). Not yet done.
4. **Deploy `cloudflared` properly** as a real Deployment in the cluster (not a one-off `kubectl run` pod like the test) — official Helm chart is the likely route. The tunnel token goes in as a Kubernetes Secret — **never commit it to this repo**. Not yet done.
5. **Map each public hostname to an internal Service** in the Cloudflare dashboard (e.g. `app.bozov-lab.com` → `http://<service>.<namespace>.svc.cluster.local:<port>`). Cloudflare auto-manages the DNS record. Not yet done.
6. Repeat step 5 per app/service that needs public exposure — one tunnel can serve multiple hostnames.

Steps 3-6 are next up, once there's an actual app worth exposing.
