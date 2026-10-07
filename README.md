# Homelab

Single-node Kubernetes homelab managed with GitOps. The baseline is
[k3s](https://k3s.io/), with [ArgoCD](https://argo-cd.readthedocs.io/) syncing
Helm charts and manifests from this repository to the cluster. Some services are
still on Docker Compose while I migrate them to k3s, see
[Services](#services) for what runs where.

## Hardware

- ASUS X550VB
- CPU: Intel i5-3230M (4) @ 3.200GHz
- GPU: NVIDIA GeForce GT 740M
- RAM: 8GB

## Stack

| Layer | Tool |
|---|---|
| OS | Ubuntu Server |
| Kubernetes | k3s (single node) |
| GitOps | ArgoCD |
| Packaging | Helm (umbrella charts) |
| Private access and DNS | Tailscale Kubernetes operator; Tailscale VPN (SSH, tailnet DNS) |
| Local load balancer | MetalLB, pool `192.168.8.96/27` (reserved for future use) |
| Legacy workloads | Docker Compose |

## Repository layout

```
apps/        ArgoCD Application manifests
charts/      Helm umbrella charts
manifests/   Plain Kubernetes manifests (MetalLB, Tailscale)
mediactr/    Media center, still on Docker Compose
legacy/      Retired stacks and archived experiments
```

### `apps/`
ArgoCD `Application` manifests. Every application visible in ArgoCD should have
a file here.

### `charts/`
Helm umbrella charts: a `Chart.yaml` whose only job is to depend on an upstream
chart, plus a `values.yaml` with my overrides. Example: `charts/adguardhome`.

### `manifests/`
Plain Kubernetes manifests, for cases where a Helm chart would be overkill.
These are currently applied by hand with `kubectl apply`, see [TODO](#todo).

- `metallb/`: `IPAddressPool` for local LoadBalancer IPs
- `tailscale/`: Tailscale `ProxyGroup` for ingress

### `mediactr/`
Media center on Docker Compose. Still in use, not yet migrated.

### `legacy/`
Not deployed and not a source of truth. Kept for reference.

| Directory | What it is |
|---|---|
| `adguardhome/` | Docker Compose AdGuard Home. Replaced by `charts/adguardhome`. |
| `zigbee2mqtt/` | Docker Compose zigbee2mqtt. Retired. |
| `postgres/` | Docker Compose PostgreSQL + pgAdmin. Retired. |
| `minikube/` | Early Kubernetes learning material (Gateway API, ConfigMaps, PVCs on Minikube). Archival only. |

## Services

| Service | Runtime | Location | Status |
|---|---|---|---|
| AdGuard Home | k3s | `charts/adguardhome` | Active |
| Jellyfin | Docker Compose | `mediactr/` | Active, migration pending |
| letterboxd-stats | Docker Compose | `mediactr/` | Broken (upstream archived) |
| zigbee2mqtt | Docker Compose | `legacy/zigbee2mqtt` | Retired |
| PostgreSQL + pgAdmin | Docker Compose | `legacy/postgres` | Retired |

### AdGuard Home
Network-wide DNS ad blocking. DNS (`:53`) is exposed on the tailnet through
Tailscale, and Tailscale uses it as the tailnet's DNS provider. The web UI is
exposed privately through the Tailscale operator
(`tailscale.com/expose: "true"`).

### Media center (`mediactr`)
Downloaded media gets subtitles from subliminal and is served by Jellyfin.
Watch history then goes to Trakt and is synced to Letterboxd by
letterboxd-stats.

```
download -> subliminal -> Jellyfin -> Trakt -> letterboxd-stats -> Letterboxd
```

- **Jellyfin**: media streaming server
- **letterboxd-stats**: syncs watch history to Letterboxd using an unofficial
  CLI client. It runs regularly from a cronjob on the host. Currently broken
  because the client repository was archived.

## Adding an application

1. Create an umbrella chart in `charts/<name>/`:
   - `Chart.yaml` with the upstream chart as a dependency
   - `values.yaml` with overrides, nested under the dependency name
2. Copy `apps/adguardhome.yaml` to `apps/<name>.yaml` and change the `name`,
   destination `namespace` and `path`.
3. Commit to `main` (or merge a pull request). The parent `apps` Application in
   ArgoCD watches `apps/` and creates the new Application, which then syncs the
   chart.

## TODO

- [ ] ArgoCD Applications for `manifests/metallb` and `manifests/tailscale`
- [ ] Bring the Tailscale exposure of AdGuard DNS under GitOps
- [ ] Encrypted secrets in git (SOPS)
- [ ] Migrate `mediactr` to k3s
