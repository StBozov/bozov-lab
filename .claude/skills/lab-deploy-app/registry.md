# App registry and resource budget

One row per app. Check it before choosing a name, hostname, port or quota, and change it in the same commit as the app. Numbers are what was measured when the budget was set; re-check live state before relying on them.

**Last confirmed against the live cluster: 2026-11-01** (update it whenever the live state is checked and the table still matches).

## Capacity

| Resource | Total | Notes |
|---|---|---|
| CPU | 12 cores | 3 workers with 4 each. `control` is tainted and runs no apps |
| Memory | about 46.8 GiB | 3 workers with about 15.6 GiB allocatable each |
| Storage | about 490 GB usable | Longhorn over the workers' NVMe drives, 2 replicas per volume (see `docs/cluster.md`) |

**Budget rule:** keep the sum of app *requests* under about 65% of the CPU and memory totals, so the remaining two workers can still run everything if one fails. Keep app storage under about 80% of usable storage. Limits may overcommit; requests may not.

| | CPU | Memory | Storage |
|---|---|---|---|
| Platform reserve (Longhorn, Traefik, Headlamp, cloudflared, DNS) | 2 | 6 GiB | n/a |
| Assignable to apps (65% of total, minus the reserve) | about 5.8 | about 24 GiB | about 390 GB |

## Apps

This table is the single source for every app's quota. An app's own skill points here instead of repeating the numbers. Status `live` means it exists on the cluster now; `planned` means the allocation is reserved but nothing is applied yet; `namespace live` means the namespace and its fencing exist but no workload runs.

| App | Status | Namespace | Hostnames | NodePorts | CPU requests / limits | Memory requests / limits | Storage | Flux path | Skill |
|---|---|---|---|---|---|---|---|---|---|
| Platform | live | `longhorn-system`, `kube-system`, `cloudflare-tunnel` | none (admin UIs on NodePorts) | 30219 (Longhorn), 30519 (Headlamp) | reserve 2 | reserve 6 GiB | n/a | n/a | `docs/cluster.md` |
| Tunnel test page | live | `tunnel-test` | `test.bozov-lab.com` | none | none set | none set | none | n/a | `docs/cloudflare-tunnel.md` |
| Crossly | namespace live | `crossly` | `crossly.bozov-lab.com` | none | 2 / 4 | 4 GiB / 8 GiB | 20 GiB in at most 6 PVCs; at most 30 pods | `apps/crossly` (planned) | `app-crossly` |
| **Unallocated** | | | | | about 3.8 | about 20 GiB | about 370 GB | | |

## Adding a row

Fill every column. A new app must not push the unallocated row below zero; if it would, either shrink another app's quota deliberately or raise the budget here first, with a reason.
