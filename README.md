# Homelab GitOps

GitOps repository for managing the homelab k3s cluster using Flux CD.

## Overview

This repository contains all Kubernetes manifests and configurations for the homelab k3s cluster. Flux CD automatically syncs these configurations to the cluster.

**GitOps Workflow:**
1. Make changes to YAML files in this repo
2. Commit and push to Git
3. Flux automatically detects changes (10m interval)
4. Flux applies changes to the cluster

## Repository Structure

```
homelab-gitops/
├── clusters/
│   └── homelab/                    # Cluster entry point
│       ├── flux-system/            # Flux CD components (auto-generated)
│       ├── sources.yaml            # Helm repositories + Git sources
│       ├── infrastructure.yaml     # Infrastructure kustomization
│       └── apps.yaml               # Application kustomization
├── infrastructure/                 # Infrastructure components
│   ├── sources/                    # Helm repositories (Bitnami, Jetstack, etc.)
│   ├── cert-manager/               # TLS certificate management
│   ├── longhorn/                   # Distributed persistent storage
│   ├── cloudflared/                # Cloudflare Tunnel (public HTTPS ingress)
│   ├── monitoring/                 # Prometheus + Grafana (disabled)
│   └── jobrecon-deps/              # JobRecon databases and brokers
└── apps/                           # Applications
    ├── pihole/                     # DNS + ad blocking
    └── jobrecon/                   # JobRecon application (Helm chart)
```

## Deployment Order

Flux deploys resources in dependency order:

```
sources (Helm repos, Git repos)
  └── infrastructure
      ├── cert-manager
      ├── longhorn
      ├── cloudflared
      └── jobrecon-deps (PostgreSQL, MongoDB, Redis, RabbitMQ, Qdrant, MinIO, Ollama)
          └── apps
              ├── pihole
              └── jobrecon
```

## Components

### Infrastructure

| Component | Purpose | Namespace | Status |
|-----------|---------|-----------|--------|
| **cert-manager** | TLS certificate management | cert-manager | Active |
| **Longhorn** | Distributed block storage | longhorn-system | Active |
| **Cloudflare Tunnel** | Public HTTPS ingress for jobrecon.se | cloudflared | Active |
| **Prometheus + Grafana** | Metrics and dashboards | monitoring | Disabled |
| **PostgreSQL** | Relational database (4 schemas) | jobrecon | Active |
| **MongoDB** | Document database | jobrecon | Active |
| **Redis** | Cache (standalone, no persistence) | jobrecon | Active |
| **RabbitMQ** | Message broker | jobrecon | Active |
| **Qdrant** | Vector database for embeddings | jobrecon | Active |
| **MinIO** | S3-compatible object storage | jobrecon | Active |
| **Ollama** | LLM inference (nomic-embed-text) | jobrecon | Active |

### Applications

| Application | Purpose | Namespace | Access |
|-------------|---------|-----------|--------|
| **Pi-hole** | Network-wide DNS + ad blocking | pihole | LoadBalancer (DNS), port-forward (UI) |
| **JobRecon** | Job matching platform | jobrecon | jobrecon.se (public), jobrecon.local (local) |

## Networking

### Public Access

Public traffic flows through Cloudflare Tunnel — no ports opened on the router:

```
Browser → https://jobrecon.se → Cloudflare Edge (TLS) → Tunnel → cloudflared pod
    → frontend (nginx) → /api/* proxied to gateway internally
```

### Local Access

Local traffic uses Traefik ingress with Pi-hole DNS:

- `jobrecon.local` → frontend (nginx)
- `api.jobrecon.local` → gateway (YARP)

Pi-hole resolves `*.jobrecon.local` to the cluster IP (192.168.1.76).

## Secrets Management

Secrets are encrypted with [SOPS](https://github.com/getsops/sops) using an age key and decrypted by Flux at deploy time.

```bash
# Encrypt a secret
sops -e -i path/to/secret.sops.yaml

# Decrypt for viewing
sops -d path/to/secret.sops.yaml
```

Encrypted secret files:
- `apps/jobrecon/secrets.sops.yaml` — JWT signing key, service passwords
- `infrastructure/jobrecon-deps/infra-secrets.sops.yaml` — database passwords
- `infrastructure/cloudflared/secret.sops.yaml` — Cloudflare Tunnel token

## Prerequisites

1. k3s cluster running (see homelab-ansible repo)
2. kubectl access configured (`~/.kube/homelab-k3s-config`)
3. SOPS + age key for secret management
4. Flux bootstrapped to this repo

## Common Operations

### Check Sync Status

```bash
kubectl get kustomizations -n flux-system
kubectl get helmreleases -A
```

### Force Sync

```bash
kubectl annotate kustomization infrastructure \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" -n flux-system

kubectl annotate kustomization apps \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" -n flux-system
```

### View Logs

```bash
# Flux logs
kubectl logs -n flux-system deploy/kustomize-controller

# Specific app
kubectl logs -n jobrecon -l app.kubernetes.io/component=frontend
```

### Troubleshooting

```bash
# Check events
kubectl get events -A --sort-by='.lastTimestamp'

# Describe a failing kustomization
kubectl describe kustomization apps -n flux-system

# Check pod status
kubectl get pods -A
```

## Cluster Recovery

If the cluster needs to be rebuilt:

1. Rebuild k3s cluster (use homelab-ansible)
2. Re-bootstrap Flux
3. Flux automatically restores all applications from Git

## Links

- [Flux CD](https://fluxcd.io/docs/)
- [k3s](https://docs.k3s.io/)
- [Longhorn](https://longhorn.io/docs/)
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [SOPS](https://github.com/getsops/sops)
