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
| Local load balancer | MetalLB, pool `192.168.8.96/27` | 
| Legacy workloads | Docker Compose |
| Secrets | SOPS + age |

## Repository layout

```
bootstrap/   Root ArgoCD bootstrap application ("app of apps")
apps/        ArgoCD Application manifests
charts/      Helm umbrella charts
manifests/   Plain Kubernetes manifests (Ingress, MetalLB, Tailscale, ArgoCD)
mediactr/    Media center, still on Docker Compose
legacy/      Retired stacks and archived experiments
```

### `bootstrap/`
Contains `bootstrap/apps.yaml`, the root ArgoCD Application implementing the
"app of apps" pattern. It monitors the `apps/` directory and creates all child
applications in the cluster.

### `apps/`
ArgoCD `Application` manifests. Every application visible in ArgoCD has a file
here. Several applications use multi-source configurations to sync both a Helm
chart from `charts/` and supplementary manifests (such as Tailscale Ingresses)
from `manifests/`.

### `charts/`
Helm umbrella charts: a `Chart.yaml` whose primary job is to depend on an upstream
chart, plus a `values.yaml` with custom overrides. Encrypted values are stored in
`secrets.yaml` alongside.

### `manifests/`
Plain Kubernetes manifests:
- `argocd/`: ArgoCD install base (upstream manifest + Kustomize patches for Helm Secrets)
- `<app>/ingress.yaml`: Tailscale Ingress resources for applications (`grafana`, `homeassistant`, `linkding`, `prometheus`)
- `metallb/`: `IPAddressPool` for local LoadBalancer IPs
- `tailscale/`: Tailscale `ProxyGroup` for ingress

### `mediactr/`
Media center on Docker Compose. Still in use, not yet migrated.

### `legacy/`
Not deployed and not a source of truth. Kept for reference.

| Directory | What it is |
|---|---|
| `adguardhome/` | Docker Compose AdGuard Home. Replaced by `charts/adguardhome`. |
| `zigbee2mqtt/` | Docker Compose zigbee2mqtt. Replaced by `charts/zigbee2mqtt`. |
| `postgres/` | Docker Compose PostgreSQL + pgAdmin. Retired. |
| `minikube/` | Early Kubernetes learning material (Gateway API, ConfigMaps, PVCs on Minikube). Archival only. |

## Services

| Service | Runtime | Location | Status |
|---|---|---|---|
| AdGuard Home | k3s | `charts/adguardhome` | Active |
| Home Assistant | k3s | `charts/homeassistant` | Active |
| Mosquitto | k3s | `charts/mosquitto` | Active |
| Zigbee2MQTT | k3s | `charts/zigbee2mqtt` | Active |
| Linkding | k3s | `charts/linkding` | Active |
| Prometheus | k3s | `charts/prometheus` | Active |
| Grafana | k3s | `charts/grafana` | Active |
| Jellyfin | Docker Compose | `mediactr/` | Active, migration pending |
| letterboxd-stats | Docker Compose | `mediactr/` | Broken (upstream archived) |
| PostgreSQL + pgAdmin | Docker Compose | `legacy/postgres` | Retired |

### AdGuard Home
Network-wide DNS ad blocking. DNS (`:53`) is exposed on the tailnet through
Tailscale, and Tailscale uses it as the tailnet's DNS provider. The web UI is
exposed privately through the Tailscale operator (`tailscale.com/expose: "true"`).

### Home Assistant
Open-source home automation platform. Exposes its web interface over Tailscale
via an Ingress using the `ingress-proxies` ProxyGroup.

### Mosquitto & Zigbee2MQTT
- **Mosquitto**: MQTT broker exposed locally via a MetalLB LoadBalancer IP for local IoT communication.
- **Zigbee2MQTT**: Bridges Zigbee devices via a connected USB coordinator directly to the Mosquitto MQTT broker, integrated with Home Assistant.

### Linkding
Lightweight self-hosted bookmark manager with tagging and archive search, exposed over Tailscale Ingress.

### Monitoring (Prometheus & Grafana)
- **Prometheus**: Scrapes and stores node and Kubernetes cluster metrics.
- **Grafana**: Visualizes metrics collected by Prometheus across pre-configured dashboards.

### Media center (`mediactr`)
Downloaded media gets subtitles from subliminal and is served by Jellyfin.
Watch history then goes to Trakt and is synced to Letterboxd by letterboxd-stats.

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
2. If custom manifests (like a Tailscale `Ingress`) are needed, add them to
   `manifests/<name>/`.
3. Copy an existing application in `apps/` (e.g. `apps/linkding.yaml`) to `apps/<name>.yaml`
   and configure the destination, chart path, and any manifest source paths.
4. If the chart needs secrets: create `charts/<name>/secrets.yaml` with
   `sops --encrypt --age <recipient> values-plain.yaml > secrets.yaml` and add
   `secrets://secrets.yaml` to `valueFiles` in the app manifest.
5. Commit to `main` (or merge a pull request). The root `apps` Application in
   ArgoCD watches `apps/` and deploys the new Application automatically.

## TODO

- [ ] ArgoCD Applications for `manifests/metallb` and `manifests/tailscale`
- [ ] Bring the Tailscale exposure of AdGuard DNS under GitOps
- [x] Encrypted secrets in git (SOPS)
- [x] ArgoCD install base under GitOps (`manifests/argocd`)
- [ ] Migrate `mediactr` to k3s
