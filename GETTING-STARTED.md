# Getting Started with Homelab GitOps

This guide will help you deploy Flux CD and all applications to your k3s cluster.

## Prerequisites

✅ **Before you begin, ensure:**

1. k3s cluster is running (3 nodes)
2. kubectl is configured (`export KUBECONFIG=~/.kube/homelab-k3s-config`)
3. You can access the cluster: `kubectl get nodes`
4. HTTP/HTTPS ports are open (80, 443) - see homelab-ansible repo

## Step 1: Install Flux CLI

### Linux/macOS

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

### Windows (PowerShell as Administrator)

```powershell
choco install flux
```

### Verify Installation

```bash
flux --version
# Should output: flux version X.X.X
```

## Step 2: Pre-flight Check

Verify your cluster is ready for Flux:

```bash
flux check --pre
```

**Expected output:**
```
► checking prerequisites
✔ Kubernetes 1.31.4+k3s1 >=1.26.0-0
✔ prerequisites checks passed
```

If you see any errors, fix them before continuing.

## Step 3: Install Flux to the Cluster

### Option A: Local Git Repository (Development)

For testing without pushing to GitHub:

```bash
# From the homelab-gitops directory
cd /path/to/homelab-gitops

# Install Flux components
flux install

# Apply the cluster configuration
kubectl apply -f clusters/homelab/sources.yaml
kubectl apply -f clusters/homelab/infrastructure.yaml
kubectl apply -f clusters/homelab/apps.yaml
```

### Option B: GitHub/GitLab (Production)

For production use with automatic sync:

**First, push this repo to GitHub:**

```bash
# Create a new GitHub repository called 'homelab-gitops'
# Then push this repo:
git remote add origin git@github.com:yourusername/homelab-gitops.git
git add .
git commit -m "initial commit: GitOps repository"
git push -u origin main
```

**Then bootstrap Flux:**

```bash
# For GitHub
flux bootstrap github \
  --owner=yourusername \
  --repository=homelab-gitops \
  --branch=main \
  --path=clusters/homelab \
  --personal

# For GitLab
flux bootstrap gitlab \
  --owner=yourusername \
  --repository=homelab-gitops \
  --branch=main \
  --path=clusters/homelab \
  --personal
```

You'll be prompted for your GitHub/GitLab token.

## Step 4: Verify Deployment

### Check Flux is Running

```bash
# Check Flux components
flux check

# Get all Flux resources
flux get all
```

**Expected output:**
```
NAME                    REVISION        SUSPENDED       READY   MESSAGE
gitrepository/flux-system main@sha1:xxxxx False          True    stored artifact

NAME                    REVISION        SUSPENDED       READY   MESSAGE
kustomization/flux-system main@sha1:xxxxx False          True    Applied revision: main@sha1:xxxxx
kustomization/sources   main@sha1:xxxxx False          True    Applied revision: main@sha1:xxxxx
kustomization/infrastructure main@sha1:xxxxx False      True    Applied revision: main@sha1:xxxxx
kustomization/apps      main@sha1:xxxxx False          True    Applied revision: main@sha1:xxxxx
```

### Watch Resources Deploy

```bash
# Watch all resources
watch kubectl get pods -A

# Or use k9s for a better view
k9s
```

**Deployment order:**
1. Flux system (flux-system namespace) - 30 seconds
2. Cert-manager (cert-manager namespace) - 1-2 minutes
3. Longhorn (longhorn-system namespace) - 2-3 minutes
4. Monitoring (monitoring namespace) - 3-5 minutes
5. Pi-hole (pihole namespace) - 1-2 minutes

**Total deployment time:** ~10-15 minutes

## Step 5: Verify Each Component

### Cert-manager

```bash
kubectl get pods -n cert-manager
```

**Expected:** 3 pods running (cert-manager, webhook, cainjector)

```bash
kubectl get clusterissuer
```

**Expected:** letsencrypt-staging and letsencrypt-prod

### Longhorn

```bash
kubectl get pods -n longhorn-system
```

**Expected:** Many pods running (manager, driver, UI, engine, replica)

```bash
kubectl get storageclass
```

**Expected:** longhorn (default)

### Monitoring

```bash
kubectl get pods -n monitoring
```

**Expected:** Multiple pods running:
- prometheus-kube-prometheus-stack-prometheus-0
- kube-prometheus-stack-operator-xxx
- grafana-xxx
- alertmanager-xxx
- node-exporter-xxx (one per node)

### Pi-hole

```bash
kubectl get pods -n pihole
```

**Expected:** 1 pod running (pihole-xxx)

```bash
kubectl get svc -n pihole
```

**Expected:** pihole-dns service with EXTERNAL-IP assigned

## Step 6: Access Applications

### Grafana (Monitoring Dashboards)

```bash
# Port-forward Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Open http://localhost:3000
- **Username:** admin
- **Password:** prom-operator (change this in production!)

**Pre-installed dashboards:**
- Kubernetes / Compute Resources / Cluster
- Kubernetes / Compute Resources / Node (Pods)
- Node Exporter / Nodes

### Prometheus (Metrics)

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Open http://localhost:9090

### AlertManager (Alerts)

```bash
# Port-forward AlertManager
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
```

Open http://localhost:9093

### Pi-hole (DNS/Ad Blocking)

```bash
# Get the Pi-hole DNS service IP
kubectl get svc -n pihole pihole-dns
# Note the EXTERNAL-IP

# Port-forward the web interface
kubectl port-forward -n pihole svc/pihole-web 8080:80
```

Open http://localhost:8080/admin
- **Password:** changeme (change this in production!)

**To use Pi-hole as your DNS server:**
1. Note the EXTERNAL-IP of the pihole-dns service
2. Configure your router or devices to use this IP as DNS server
3. All DNS queries will now be filtered for ads

### Longhorn UI

```bash
# Port-forward Longhorn UI
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
```

Open http://localhost:8080

View:
- Volume status
- Node disk usage
- Replica health
- Backups (when configured)

## Step 7: Configure Ingress (Optional)

To access services via domain names instead of port-forward, configure ingress.

**Edit the helmrelease files to enable ingress:**

1. **Grafana:** `infrastructure/monitoring/helmrelease.yaml`
2. **Longhorn:** `infrastructure/longhorn/helmrelease.yaml`
3. **Pi-hole:** `apps/pihole/helmrelease.yaml`

Set `ingress.enabled: true` and configure hosts.

**Add DNS entries:**

Point your domains to any cluster node IP (ServiceLB will route):
```
grafana.homelab.local    → 192.168.1.76
longhorn.homelab.local   → 192.168.1.76
pihole.homelab.local     → 192.168.1.76
```

**Test:**
```bash
curl -k https://grafana.homelab.local
```

## Troubleshooting

### Flux isn't syncing

```bash
# Check Flux logs
flux logs --level=error --all-namespaces

# Force reconciliation
flux reconcile source git flux-system
flux reconcile kustomization infrastructure
flux reconcile kustomization apps
```

### Pod stuck in Pending

```bash
# Check events
kubectl describe pod <pod-name> -n <namespace>

# Common causes:
# - Insufficient resources
# - PVC not bound (Longhorn not ready)
# - Image pull error
```

### Longhorn volumes not creating

```bash
# Check Longhorn manager logs
kubectl logs -n longhorn-system -l app=longhorn-manager

# Check node disk space
kubectl get nodes -o custom-columns=NAME:.metadata.name,DISK:.status.capacity.ephemeral-storage
```

### Cert-manager certificates not issuing

```bash
# Check certificate status
kubectl get certificate -A
kubectl describe certificate <cert-name> -n <namespace>

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager
```

## Next Steps

After everything is deployed:

1. **Change default passwords:**
   - Grafana: Edit `infrastructure/monitoring/helmrelease.yaml`
   - Pi-hole: Edit `apps/pihole/helmrelease.yaml`
   - Commit and push (Flux will update automatically)

2. **Configure TLS certificates:**
   - Test with letsencrypt-staging first
   - Switch to letsencrypt-prod when working

3. **Add more applications:**
   - Create new directory in `apps/`
   - Add Kubernetes manifests
   - Update `apps/kustomization.yaml`
   - Commit and push

4. **Set up monitoring alerts:**
   - Configure AlertManager
   - Add Slack/email notifications
   - Create custom Prometheus rules

5. **Configure backups:**
   - Longhorn S3/NFS backups
   - Regular snapshots
   - Disaster recovery plan

## Useful Commands

```bash
# Watch Flux reconciliation
flux get sources git --watch

# Check all kustomizations
flux get kustomizations

# Suspend/Resume automatic sync
flux suspend kustomization apps
flux resume kustomization apps

# Uninstall Flux (careful!)
flux uninstall

# Export current state
flux export source git flux-system
flux export kustomization flux-system
```

## Resources

- **Flux Documentation:** https://fluxcd.io/docs/
- **Longhorn Documentation:** https://longhorn.io/docs/
- **Cert-manager Documentation:** https://cert-manager.io/docs/
- **Prometheus Operator:** https://prometheus-operator.dev/
- **Pi-hole Documentation:** https://docs.pi-hole.net/

---

**Remember:** With GitOps, your Git repository is the source of truth. All changes should be made in Git and committed. Flux will automatically sync to the cluster.

**Happy GitOpsing! 🚀**
