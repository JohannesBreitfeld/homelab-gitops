# Homelab GitOps

GitOps repository for managing the homelab Kubernetes cluster using Flux CD.

## Overview

This repository contains all Kubernetes manifests and configurations for the homelab k3s cluster. Flux CD automatically syncs these configurations to the cluster.

**GitOps Workflow:**
1. Make changes to YAML files in this repo
2. Commit and push to Git
3. Flux automatically detects changes
4. Flux applies changes to the cluster

## Repository Structure

```
homelab-gitops/
├── clusters/
│   └── homelab/                    # Homelab cluster configuration
│       ├── flux-system/            # Flux CD components (auto-generated)
│       ├── infrastructure.yaml     # Infrastructure apps kustomization
│       └── apps.yaml               # User apps kustomization
├── infrastructure/                 # Infrastructure components
│   ├── sources/                    # Helm repositories
│   ├── cert-manager/               # TLS certificate management
│   ├── longhorn/                   # Persistent storage
│   └── monitoring/                 # Prometheus + Grafana
└── apps/                           # Applications
    └── pihole/                     # DNS + Ad blocking
```

## Components

### Infrastructure

| Component | Purpose | Namespace |
|-----------|---------|-----------|
| **cert-manager** | Automatic TLS certificates from Let's Encrypt | cert-manager |
| **Longhorn** | Distributed block storage for persistent volumes | longhorn-system |
| **Prometheus** | Metrics collection and alerting | monitoring |
| **Grafana** | Metrics visualization and dashboards | monitoring |

### Applications

| Application | Purpose | Namespace |
|-------------|---------|-----------|
| **Pi-hole** | Network-wide ad blocking and DNS | pihole |

## Prerequisites

Before deploying:

1. ✅ k3s cluster running (see homelab-ansible repo)
2. ✅ kubectl access configured
3. ✅ Flux CLI installed (see below)

## Quick Start

### 1. Install Flux CLI

**Linux/macOS:**
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

**Windows (PowerShell):**
```powershell
choco install flux
```

**Verify installation:**
```bash
flux --version
```

### 2. Bootstrap Flux

From this repository root:

```bash
# Set your cluster kubeconfig
export KUBECONFIG=~/.kube/homelab-k3s-config

# Check cluster is accessible
kubectl get nodes

# Bootstrap Flux (installs Flux to the cluster)
flux bootstrap git \
  --url=ssh://git@github.com/yourusername/homelab-gitops \
  --branch=main \
  --path=clusters/homelab

# Or for local development (no Git push required)
flux install
kubectl apply -f clusters/homelab/infrastructure.yaml
kubectl apply -f clusters/homelab/apps.yaml
```

### 3. Verify Deployment

```bash
# Check Flux is running
flux check

# Watch Flux sync status
flux get all

# Check infrastructure deployments
kubectl get kustomizations -A

# Check all pods
kubectl get pods -A
```

## Managing Applications

### Adding a New Application

1. Create directory: `apps/myapp/`
2. Add Kubernetes manifests (Deployment, Service, Ingress, etc.)
3. Create `apps/myapp/kustomization.yaml`
4. Add to `clusters/homelab/apps.yaml`
5. Commit and push - Flux will deploy automatically

### Updating an Application

1. Edit the manifest files
2. Commit and push
3. Flux automatically applies changes (within 1 minute)

### Force Sync

If you don't want to wait for automatic sync:

```bash
# Reconcile specific kustomization
flux reconcile kustomization infrastructure -n flux-system

# Reconcile apps
flux reconcile kustomization apps -n flux-system
```

## Common Operations

### Check Sync Status

```bash
# All kustomizations
flux get kustomizations

# Specific app
flux get kustomization cert-manager -n flux-system
```

### View Logs

```bash
# Flux controller logs
flux logs --level=error --all-namespaces

# Specific deployment
kubectl logs -n monitoring -l app=prometheus
```

### Suspend/Resume Sync

```bash
# Suspend automatic sync (for maintenance)
flux suspend kustomization apps -n flux-system

# Resume
flux resume kustomization apps -n flux-system
```

### Troubleshooting

```bash
# Check Flux health
flux check

# Get events
kubectl get events -A --sort-by='.lastTimestamp'

# Describe kustomization for errors
kubectl describe kustomization apps -n flux-system
```

## Repository Guidelines

### Commit Message Format

```
<type>: <description>

Types:
- feat: New feature or application
- fix: Bug fix
- update: Update existing resource
- remove: Remove resource
- docs: Documentation changes
```

**Examples:**
```
feat: add pihole DNS server
update: increase grafana resources
fix: correct cert-manager ClusterIssuer
```

### YAML Best Practices

1. **Use comments**: Explain non-obvious configurations
2. **Pin versions**: Specify exact image tags, not `latest`
3. **Set resource limits**: Define CPU/memory requests and limits
4. **Use namespaces**: Don't use `default` namespace
5. **Label everything**: Add consistent labels for organization

### Security

- **Never commit secrets**: Use Sealed Secrets or external-secrets
- **Use RBAC**: Define minimal permissions
- **Network policies**: Restrict pod-to-pod communication where needed
- **Image policies**: Use trusted registries only

## Accessing Services

### Via kubectl port-forward

```bash
# Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80

# Pi-hole admin
kubectl port-forward -n pihole svc/pihole-web 8080:80
```

### Via Ingress (when configured)

After cert-manager and ingress are set up:
- Grafana: https://grafana.homelab.local
- Pi-hole: https://pihole.homelab.local

## Backup and Disaster Recovery

### Backup This Repository

```bash
# This Git repository IS your backup
# Push to GitHub/GitLab for redundancy
git remote add origin git@github.com:yourusername/homelab-gitops.git
git push -u origin main
```

### Cluster Recovery

If the cluster needs to be rebuilt:

1. Rebuild k3s cluster (use homelab-ansible)
2. Re-run Flux bootstrap
3. Flux automatically restores all applications from Git

This is the power of GitOps - your Git repo is the source of truth!

## Monitoring

### Metrics

Access Grafana dashboards:
```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
# Open http://localhost:3000
# Default login: admin/prom-operator
```

**Pre-installed Dashboards:**
- Kubernetes cluster overview
- Node metrics
- Pod metrics
- Persistent volume usage

### Alerts

Prometheus AlertManager:
```bash
kubectl port-forward -n monitoring svc/alertmanager 9093:9093
# Open http://localhost:9093
```

## Maintenance

### Update Flux

```bash
# Check for updates
flux check --pre

# Update Flux components
flux install --export > clusters/homelab/flux-system/gotk-components.yaml

# Commit and push
git add clusters/homelab/flux-system/
git commit -m "update: flux components to vX.X.X"
git push
```

### Update Application

Edit version in the respective HelmRelease or Deployment, commit and push.

## Links

- **Flux CD Docs**: https://fluxcd.io/docs/
- **k3s Docs**: https://docs.k3s.io/
- **Cert-manager Docs**: https://cert-manager.io/docs/
- **Longhorn Docs**: https://longhorn.io/docs/
- **Grafana Dashboards**: https://grafana.com/grafana/dashboards/

## Support

For issues with:
- **Infrastructure (nodes, k3s)**: See `homelab-ansible` repository
- **Applications (deployments, configs)**: Check this repository's issues

---

**GitOps Philosophy**: "If it's not in Git, it doesn't exist." - Everything running in the cluster should be defined in this repository.
