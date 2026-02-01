# 1Password SDK Provider Migration

## Context

User has a 1Password Service Account token (`ops_...`) which works with the **1Password SDK provider**.
The existing k3s-coolkid-homelab plan assumed 1Password Connect Server (deprecated, more complex).

**Decision**: Migrate to SDK provider for simpler homelab setup.

---

## Changes Required

### 1. Update ClusterSecretStore Configuration

**File**: `infrastructure/external-secrets/secret-store.yaml`

**Current** (Connect Server approach):
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: onepassword
spec:
  provider:
    onepassword:
      connectHost: http://onepassword-connect.external-secrets.svc.cluster.local:8080
      vaults:
        homelab: 1
      auth:
        secretRef:
          connectTokenSecretRef:
            name: onepassword-connect-token
            namespace: external-secrets
            key: token
```

**NEW** (SDK provider approach):
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: onepassword
spec:
  provider:
    onepassword:
      # Using 1Password SDK provider (no Connect Server needed)
      vaults:
        homelab: 1  # Update with actual vault ID if different
      auth:
        secretRef:
          serviceAccountTokenSecretRef:
            name: onepassword-service-account-token
            namespace: external-secrets
            key: token
```

---

### 2. Remove 1Password Connect Server

**Delete directory**: `infrastructure/1password-connect/` (already empty)

**Remove from root app**: Edit `bootstrap/argocd/root-app.yaml` to remove 1password-connect Application reference (if present)

---

### 3. Update Bootstrap Secrets Instructions

**File**: `bootstrap/secrets/README.md`

**Update Section 1** to use Service Account token:

```markdown
## 1. 1Password Service Account Token

You have a Service Account token (starts with `ops_`). Create the secret:

```bash
kubectl create namespace external-secrets

kubectl create secret generic onepassword-service-account-token \
  --from-file=token=/home/seeg/infra/kube/1password-credentials.json \
  -n external-secrets
```

**Or using literal**:
```bash
kubectl create secret generic onepassword-service-account-token \
  --from-literal=token=$(cat /home/seeg/infra/kube/1password-credentials.json) \
  -n external-secrets
```
```

---

## Immediate Actions (Manual)

Since ArgoCD is already installed, you can proceed with creating secrets:

### Step 1: Create 1Password Secret

```bash
# Create namespace
kubectl create namespace external-secrets

# Create secret from your token file
kubectl create secret generic onepassword-service-account-token \
  --from-literal=token=$(cat /home/seeg/infra/kube/1password-credentials.json) \
  -n external-secrets

# Verify
kubectl get secret onepassword-service-account-token -n external-secrets
```

### Step 2: Create Cloudflare Secrets

```bash
# Create namespaces
kubectl create namespace cert-manager
kubectl create namespace external-dns

# Create Cloudflare API token secrets
# Replace YOUR_CLOUDFLARE_API_TOKEN with actual token
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=YOUR_CLOUDFLARE_API_TOKEN \
  -n cert-manager

kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=YOUR_CLOUDFLARE_API_TOKEN \
  -n external-dns

# Verify
kubectl get secret cloudflare-api-token -n cert-manager
kubectl get secret cloudflare-api-token -n external-dns
```

### Step 3: Update secret-store.yaml

```bash
cd /home/seeg/infra/kube

# Backup current file
cp infrastructure/external-secrets/secret-store.yaml infrastructure/external-secrets/secret-store.yaml.backup

# Update the file (manual edit or use sed)
cat > infrastructure/external-secrets/secret-store.yaml <<'EOF'
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: onepassword
spec:
  provider:
    onepassword:
      vaults:
        homelab: 1
      auth:
        secretRef:
          serviceAccountTokenSecretRef:
            name: onepassword-service-account-token
            namespace: external-secrets
            key: token
EOF
```

### Step 4: Remove 1password-connect from Root App (if present)

```bash
# Check if it's referenced in root app
grep -r "1password-connect" bootstrap/argocd/

# If found, remove those lines from the root app manifest
```

### Step 5: Commit Changes

```bash
git add infrastructure/external-secrets/secret-store.yaml
git add bootstrap/secrets/README.md
git commit -m "refactor: migrate from 1Password Connect to SDK provider

- Update ClusterSecretStore to use serviceAccountTokenSecretRef
- Remove Connect Server dependency (simpler homelab setup)
- Update bootstrap secrets documentation"

git push
```

### Step 6: Deploy Root App

```bash
kubectl apply -f bootstrap/argocd/root-app.yaml

# Watch ArgoCD sync
kubectl get applications -n argocd -w
```

---

## Verification

```bash
# Check ClusterSecretStore status
kubectl get clustersecretstore onepassword

# Check External Secrets Operator logs
kubectl logs -n external-secrets deployment/external-secrets

# Test with a sample ExternalSecret
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: test-secret
  namespace: default
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: onepassword
  target:
    name: test-secret
  data:
    - secretKey: test
      remoteRef:
        key: <your-test-item-name>
        property: password
EOF

# Check if secret was created
kubectl get secret test-secret -n default
```

---

## Benefits of SDK Provider

- ✅ **No Connect Server to deploy** - Reduces cluster resource usage
- ✅ **Simpler setup** - One secret instead of credentials + token
- ✅ **Direct API calls** - No intermediate service
- ✅ **Officially recommended** - ESO docs prefer SDK over Connect
- ✅ **Perfect for homelab** - Lower overhead for small deployments

---

## Rollback (If Needed)

If SDK provider doesn't work, you can revert to Connect Server approach:

1. Restore `secret-store.yaml.backup`
2. Deploy 1Password Connect Server
3. Create `1password-credentials.json` and access token secrets
4. Update root app to include Connect Server Application
