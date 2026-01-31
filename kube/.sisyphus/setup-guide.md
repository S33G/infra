# K3s Homelab GitOps Setup

## Overview

This guide walks through setting up a complete GitOps-managed K3s homelab cluster with:
- ArgoCD for GitOps deployments
- 1Password SDK for secrets management
- Cloudflare for DNS and TLS certificates
- MetalLB, Longhorn, monitoring, and more

---

## Prerequisites

- K3s cluster running (installed on deathstar.home)
- kubectl access configured from your workstation
- 1Password Service Account token
- Cloudflare API token with DNS edit permissions
- Git repository: https://github.com/seeg/infra.git

---

## Step 1: Configure Kubectl Access

### On the K3s server (deathstar.home)

```bash
# Create kubeconfig directory
mkdir -p ~/.kube

# Copy K3s kubeconfig to user directory
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

# Change ownership
sudo chown $USER:$USER ~/.kube/config

# Secure the file
chmod 600 ~/.kube/config

# Update server address (use short hostname for cert compatibility)
sed -i 's/127.0.0.1/deathstar/g' ~/.kube/config

# Set KUBECONFIG environment variable
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc

# Test kubectl access
kubectl get nodes
```

### On your workstation (c3po)

```bash
# Create kubeconfig directory
mkdir -p ~/.kube

# Copy kubeconfig from deathstar
scp deathstar.home:~/.kube/config ~/.kube/config
chmod 600 ~/.kube/config

# Set KUBECONFIG environment variable
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc

# Ensure 'deathstar' hostname resolves
ping -c 2 deathstar

# Test kubectl access
kubectl get nodes
```

**Expected output**: Should show `deathstar` node in Ready state.

---

## Step 2: Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready (2-3 minutes)
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

**Save the admin password!** You'll need it to access ArgoCD UI.

### Access ArgoCD UI

**Option 1: Port Forward (Quick Access)**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Then open: `https://localhost:8080`
- Username: `admin`
- Password: (from command above)

**Option 2: Via Ingress (After full setup)**
ArgoCD will be available at: `https://argocd.k3s.s33g.uk`

---

## Step 3: Create Bootstrap Secrets

### 3.1 Create Namespaces

```bash
kubectl create namespace external-secrets
kubectl create namespace cert-manager
kubectl create namespace external-dns
```

### 3.2 Create 1Password Service Account Secret

You should have a file `1password-credentials.json` containing your Service Account token (starts with `ops_`).

```bash
# Create secret from your token file
kubectl create secret generic onepassword-service-account-token \
  --from-literal=token=$(cat /home/seeg/infra/kube/1password-credentials.json) \
  -n external-secrets

# Verify
kubectl get secret onepassword-service-account-token -n external-secrets
```

### 3.3 Create Cloudflare API Token Secrets

**First, get your Cloudflare API token:**
1. Go to Cloudflare dashboard → API Tokens → Create Token
2. Use "Edit zone DNS" template
3. Permissions: Zone:Zone:Read, Zone:DNS:Edit
4. Zone Resources: Include → Specific zone → s33g.uk

**Then create the secrets:**

```bash
# Replace YOUR_CLOUDFLARE_API_TOKEN with your actual token
export CLOUDFLARE_TOKEN="your-api-token-here"

kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=$CLOUDFLARE_TOKEN \
  -n cert-manager

kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=$CLOUDFLARE_TOKEN \
  -n external-dns

# Verify
kubectl get secret cloudflare-api-token -n cert-manager
kubectl get secret cloudflare-api-token -n external-dns
```

---

## Step 4: Update Configuration for 1Password SDK Provider

The repository was initially configured for 1Password Connect Server, but we're using the simpler SDK provider instead.

```bash
cd /home/seeg/infra/kube

# Backup current SecretStore configuration
cp infrastructure/external-secrets/secret-store.yaml infrastructure/external-secrets/secret-store.yaml.backup

# Update to use SDK provider (no Connect Server needed)
cat > infrastructure/external-secrets/secret-store.yaml <<'EOF'
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: onepassword
spec:
  provider:
    onepassword:
      vaults:
        homelab: 1  # Update with your actual vault ID if different
      auth:
        secretRef:
          serviceAccountTokenSecretRef:
            name: onepassword-service-account-token
            namespace: external-secrets
            key: token
EOF

# Commit the change
git add infrastructure/external-secrets/secret-store.yaml
git commit -m "refactor: migrate to 1Password SDK provider

- Use serviceAccountTokenSecretRef for simpler setup
- Remove Connect Server dependency
- Better suited for homelab use case"

git push
```

---

## Step 5: Deploy Root App-of-Apps

The root application will trigger ArgoCD to automatically deploy all infrastructure components.

```bash
# Apply the root application
kubectl apply -f bootstrap/argocd/root-app.yaml

# Watch ArgoCD sync all applications
kubectl get applications -n argocd

# Watch in real-time
kubectl get applications -n argocd -w
```

**What gets deployed:**
- MetalLB (LoadBalancer IP management)
- Longhorn (Persistent storage)
- cert-manager (TLS certificates)
- External Secrets Operator (Secrets sync from 1Password)
- ExternalDNS (Cloudflare DNS automation)
- Monitoring stack (Prometheus + Grafana)
- Ingress routes for services

---

## Step 6: Verify Deployment

### Check ArgoCD Applications

```bash
# Check all applications are synced
kubectl get applications -n argocd

# Check specific application details
kubectl describe application metallb -n argocd
```

**Expected status**: All applications should show `Synced` and `Healthy`.

### Check All Pods

```bash
# Check all namespaces
kubectl get pods -A

# Check specific namespaces
kubectl get pods -n metallb-system
kubectl get pods -n longhorn-system
kubectl get pods -n cert-manager
kubectl get pods -n external-secrets
kubectl get pods -n monitoring
```

### Check ClusterSecretStore

```bash
# Verify 1Password ClusterSecretStore is ready
kubectl get clustersecretstore onepassword

# Should show Ready status
kubectl describe clustersecretstore onepassword
```

### Check Ingress Routes

```bash
# Check Traefik IngressRoutes
kubectl get ingressroute -A

# Check LoadBalancer services
kubectl get svc -A | grep LoadBalancer
```

---

## Step 7: Access Services

Once everything is synced and healthy:

### ArgoCD UI
- URL: `https://argocd.k3s.s33g.uk`
- Username: `admin`
- Password: (from Step 2)

### Grafana Dashboard
- URL: `https://grafana.k3s.s33g.uk`
- Default credentials: Check monitoring configuration

### Longhorn UI
- URL: `https://longhorn.k3s.s33g.uk`
- Storage management interface

---

## Troubleshooting

### ArgoCD Application Not Syncing

```bash
# Check application status
kubectl describe application <app-name> -n argocd

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-application-controller

# Manual sync
kubectl patch application <app-name> -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

### ClusterSecretStore Not Ready

```bash
# Check External Secrets Operator logs
kubectl logs -n external-secrets deployment/external-secrets

# Verify secret exists
kubectl get secret onepassword-service-account-token -n external-secrets

# Check secret content (should show token key)
kubectl describe secret onepassword-service-account-token -n external-secrets
```

### Certificates Not Issuing

```bash
# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Check certificate status
kubectl get certificate -A
kubectl describe certificate <cert-name> -n <namespace>

# Check Cloudflare secret
kubectl get secret cloudflare-api-token -n cert-manager
```

### DNS Records Not Creating

```bash
# Check ExternalDNS logs
kubectl logs -n external-dns deployment/external-dns

# Verify Cloudflare token has correct permissions
kubectl describe secret cloudflare-api-token -n external-dns
```

### MetalLB Not Assigning IPs

```bash
# Check MetalLB logs
kubectl logs -n metallb-system deployment/controller
kubectl logs -n metallb-system daemonset/speaker

# Check IPAddressPool configuration
kubectl get ipaddresspool -n metallb-system
kubectl describe ipaddresspool -n metallb-system
```

---

## Maintenance

### Update ArgoCD

```bash
# Update to latest stable version
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Update Application Configurations

1. Make changes to YAML files in the repository
2. Commit and push to Git
3. ArgoCD will automatically detect and sync changes (default: 3 minutes)
4. Or manually sync via ArgoCD UI or CLI

### Add New Applications

1. Create application directory under `apps/` or `infrastructure/`
2. Add `application.yaml` with ArgoCD Application manifest
3. Reference in root app if needed
4. Commit and push
5. Apply or wait for ArgoCD to sync

---

## Architecture Overview

```
K3s Cluster (deathstar)
├── ArgoCD (GitOps Controller)
│   └── Root App-of-Apps
│       ├── Infrastructure Apps
│       │   ├── MetalLB (LoadBalancer)
│       │   ├── Longhorn (Storage)
│       │   ├── cert-manager (TLS)
│       │   ├── External Secrets Operator
│       │   ├── ExternalDNS (Cloudflare)
│       │   └── Monitoring (Prometheus/Grafana)
│       └── Application Apps (future)
│
├── Secrets (Bootstrap)
│   ├── 1Password Service Account Token
│   └── Cloudflare API Tokens
│
└── Ingress (Traefik)
    └── Routes for all services
```

---

## Network Configuration

- **Cluster CIDR**: `10.200.0.0/16`
- **LoadBalancer IP Pool**: `10.200.100.1-30` (MetalLB L2)
- **Domain**: `k3s.s33g.uk`
- **Services**: `*.k3s.s33g.uk`
- **DNS Provider**: Cloudflare
- **TLS Provider**: Let's Encrypt (via cert-manager)

---

## Security Notes

1. **ArgoCD Admin Password**: Change the default admin password after first login
2. **Secrets**: All secrets managed via 1Password + External Secrets Operator
3. **TLS**: All services use valid Let's Encrypt certificates
4. **Network**: MetalLB L2 mode for LoadBalancer services
5. **RBAC**: ArgoCD manages cluster resources with appropriate permissions

---

## Reference Documentation

- **ArgoCD**: https://argo-cd.readthedocs.io/
- **External Secrets Operator**: https://external-secrets.io/
- **1Password SDK Provider**: https://external-secrets.io/latest/provider/1password-sdk/
- **cert-manager**: https://cert-manager.io/
- **MetalLB**: https://metallb.universe.tf/
- **Longhorn**: https://longhorn.io/
- **K3s**: https://k3s.io/

---

## Additional Resources

- **Main Plan**: `.sisyphus/plans/k3s-coolkid-homelab.md`
- **Migration Guide**: `.sisyphus/plans/1password-sdk-migration.md`
- **Bootstrap Secrets**: `bootstrap/secrets/README.md`
- **Repository**: https://github.com/seeg/infra
