# ArgoCD Sync Wave Fix

## Problem

ArgoCD root app is discovering and trying to apply configuration YAMLs (like MetalLB IPAddressPool, cert-manager ClusterIssuer) **before** the operators that install those CRDs are deployed.

**Error messages:**
```
The Kubernetes API could not find metallb.io/IPAddressPool
Make sure the "IPAddressPool" CRD is installed on the destination cluster.
```

## Root Cause

The root-app.yaml uses a directory include pattern that matches ALL yaml files:
```yaml
include: '{infrastructure/**/*.yaml,ingress/**/*.yaml}'
```

This causes it to apply both:
1. Application manifests (which install operators)
2. Configuration files (which depend on CRDs from those operators)

...at the same time, causing failures.

## Solution Options

### Option 1: Use Sync Waves (Recommended - Minimal Changes)

Add sync wave annotations to control order:
- Wave 0: Operator Applications (metallb, cert-manager, external-secrets, etc.)
- Wave 1: Configurations (IPAddressPool, ClusterIssuers, etc.)

### Option 2: Restructure Directory (Better Long-term)

Separate Application manifests from configurations:
```
infrastructure/
├── applications/          # ArgoCD Application manifests only
│   ├── metallb.yaml
│   ├── cert-manager.yaml
│   └── ...
├── metallb/              # MetalLB configs (deployed by metallb app)
│   ├── ip-pool.yaml
│   └── l2-advertisement.yaml
├── cert-manager/         # cert-manager configs (deployed by cert-manager app)
│   └── issuers/
└── ...
```

Root app only includes `applications/**/*.yaml`.

Each Application then points to its own config directory.

---

## Quick Fix: Add Sync Waves

### Step 1: Update Application Manifests (Wave 0)

Add sync wave annotation to all operator applications:

```bash
cd /home/seeg/infra/kube

# Add sync wave to metallb application
cat > infrastructure/metallb/application.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metallb
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"  # Deploy operators first
spec:
  project: default
  source:
    repoURL: https://metallb.github.io/metallb
    chart: metallb
    targetRevision: 0.14.9
  destination:
    server: https://kubernetes.default.svc
    namespace: metallb-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

# Similar for cert-manager, external-secrets, longhorn, external-dns
# Add: annotations: { argocd.argoproj.io/sync-wave: "0" }
```

### Step 2: Update Configuration Files (Wave 1)

Add sync wave to configuration resources:

```bash
# Update MetalLB config
cat > infrastructure/metallb/config.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: metallb-config-placeholder
  namespace: metallb-system
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Deploy configs after operators
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  addresses:
    - 10.200.100.1-10.200.100.30
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  ipAddressPools:
    - homelab-pool
EOF

# Similar for cert-manager ClusterIssuers, External Secrets ClusterSecretStore
```

### Step 3: Commit and Push

```bash
git add infrastructure/
git commit -m "fix: add ArgoCD sync waves for proper deployment order

- Wave 0: Operator applications (install CRDs)
- Wave 1: Configurations (depend on CRDs)
- Prevents 'CRD not found' sync failures"
git push
```

### Step 4: Trigger Sync

```bash
kubectl patch application root -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

---

## Alternative Quick Fix: Exclude Config Files from Root App

Instead of sync waves, exclude config files from root app and let Applications manage them:

### Update root-app.yaml

```yaml
spec:
  source:
    directory:
      recurse: true
      include: '{infrastructure/**/application.yaml,ingress/**/*.yaml}'  # Only application.yaml files
```

This way root app only creates Application resources, and each Application manages its own configs.

### Update Each Application to Include Configs

Example for MetalLB:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metallb
  namespace: argocd
spec:
  project: default
  sources:  # Multiple sources
    - repoURL: https://metallb.github.io/metallb
      chart: metallb
      targetRevision: 0.14.9
    - repoURL: git@github.com:s33g/infra.git
      targetRevision: HEAD
      path: kube/infrastructure/metallb
      directory:
        include: 'config.yaml'  # Include config after operator is ready
  destination:
    server: https://kubernetes.default.svc
    namespace: metallb-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    syncWaves:
      - wave: 0
        sources: [0]  # Install chart first
      - wave: 1
        sources: [1]  # Apply configs after
```

---

## Recommended Approach

**Use sync waves (Option 1)** - it's the simplest fix with minimal restructuring.

The sync wave numbers determine order:
- Wave 0 (or negative): Infrastructure operators
- Wave 1: Operator configurations
- Wave 2: Applications
- Wave 3: Ingress routes

ArgoCD waits for each wave to be Healthy before proceeding to the next.

---

## Verification After Fix

```bash
# Watch applications sync
kubectl get applications -n argocd -w

# Check sync waves are being respected
kubectl get application metallb -n argocd -o jsonpath='{.metadata.annotations}'

# Verify all apps synced and healthy
kubectl get applications -n argocd
```

Expected result:
- All applications show `Synced` and `Healthy`
- No more "CRD not found" errors
- Infrastructure deploys in correct order
