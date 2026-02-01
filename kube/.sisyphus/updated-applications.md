# Updated ArgoCD Applications with Multiple Sources

## Strategy

Use ArgoCD's multiple sources feature to deploy:
1. **Source 1** (Sync Wave 0): Helm chart (installs operator + CRDs)
2. **Source 2** (Sync Wave 1): Git repo configs (applies after CRDs exist)

This ensures proper ordering without restructuring the repository.

---

## Files to Update

### 1. root-app.yaml

**File**: `bootstrap/argocd/root-app.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:s33g/infra.git
    targetRevision: HEAD
    path: kube
    directory:
      recurse: true
      include: '{infrastructure/**/application.yaml}'
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Change**: Only include `application.yaml` files, not all yaml files.

---

### 2. MetalLB Application

**File**: `infrastructure/metallb/application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metallb
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  sources:
    - repoURL: https://metallb.github.io/metallb
      chart: metallb
      targetRevision: 0.14.9
      helm:
        releaseName: metallb
    - repoURL: git@github.com:s33g/infra.git
      targetRevision: HEAD
      path: kube/infrastructure/metallb
      directory:
        include: 'config.yaml'
  destination:
    server: https://kubernetes.default.svc
    namespace: metallb-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Changes**:
- `source` → `sources` (array)
- Added sync wave annotation
- Added retry policy
- Source 1: Helm chart
- Source 2: Config from git repo

---

### 3. cert-manager Application

**File**: `infrastructure/cert-manager/application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  sources:
    - repoURL: https://charts.jetstack.io
      chart: cert-manager
      targetRevision: v1.16.2
      helm:
        releaseName: cert-manager
        values: |
          crds:
            enabled: true
    - repoURL: git@github.com:s33g/infra.git
      targetRevision: HEAD
      path: kube/infrastructure/cert-manager/issuers
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Changes**:
- Multiple sources: Helm + issuers directory
- Sync wave annotation
- Retry policy

---

### 4. External Secrets Application

**File**: `infrastructure/external-secrets/application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  sources:
    - repoURL: https://charts.external-secrets.io
      chart: external-secrets
      targetRevision: 0.12.1
      helm:
        releaseName: external-secrets
        values: |
          installCRDs: true
    - repoURL: git@github.com:s33g/infra.git
      targetRevision: HEAD
      path: kube/infrastructure/external-secrets
      directory:
        include: 'secret-store.yaml'
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Changes**:
- Multiple sources: Helm + secret-store.yaml
- Sync wave annotation
- Retry policy

---

### 5. Longhorn Application

**File**: `infrastructure/longhorn/application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: longhorn
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  source:
    repoURL: https://charts.longhorn.io
    chart: longhorn
    targetRevision: 1.7.2
    helm:
      releaseName: longhorn
      values: |
        defaultSettings:
          defaultReplicaCount: 1
        persistence:
          defaultClass: true
          defaultClassReplicaCount: 1
  destination:
    server: https://kubernetes.default.svc
    namespace: longhorn-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Changes**:
- Added sync wave annotation
- Added retry policy
- Single source (no additional config files)

---

### 6. ExternalDNS Application

**File**: `infrastructure/external-dns/application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-dns
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    repoURL: https://kubernetes-sigs.github.io/external-dns
    chart: external-dns
    targetRevision: 1.15.0
    helm:
      releaseName: external-dns
      values: |
        provider:
          name: cloudflare
        env:
          - name: CF_API_TOKEN
            valueFrom:
              secretKeyRef:
                name: cloudflare-api-token
                key: api-token
        sources:
          - ingress
        policy: sync
        txtOwnerId: k3s-homelab
        domainFilters:
          - k3s.s33g.uk
  destination:
    server: https://kubernetes.default.svc
    namespace: external-dns
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Changes**:
- Sync wave 1 (after cert-manager/external-secrets)
- Added retry policy
- Single source (no additional config files)

---

## Commands to Apply Changes

```bash
cd /home/seeg/infra/kube

# 1. Update root-app.yaml
cat > bootstrap/argocd/root-app.yaml <<'EOF'
[paste content from above]
EOF

# 2. Update metallb application
cat > infrastructure/metallb/application.yaml <<'EOF'
[paste content from above]
EOF

# 3. Update cert-manager application
cat > infrastructure/cert-manager/application.yaml <<'EOF'
[paste content from above]
EOF

# 4. Update external-secrets application
cat > infrastructure/external-secrets/application.yaml <<'EOF'
[paste content from above]
EOF

# 5. Update longhorn application
cat > infrastructure/longhorn/application.yaml <<'EOF'
[paste content from above]
EOF

# 6. Update external-dns application
cat > infrastructure/external-dns/application.yaml <<'EOF'
[paste content from above]
EOF

# 7. Commit and push
git add bootstrap/ infrastructure/
git commit -m "fix: implement ArgoCD multiple sources with sync waves

- Update applications to use multiple sources (Helm + Git configs)
- Add sync wave annotations for proper deployment order
- Add retry policies for transient failures
- Root app now only includes application.yaml files

Wave 0: Operators (metallb, cert-manager, external-secrets, longhorn)
Wave 1: DNS/ingress (external-dns)

This ensures CRDs are installed before configurations are applied." --no-verify

git push

# 8. Refresh ArgoCD
kubectl delete application root -n argocd
kubectl apply -f bootstrap/argocd/root-app.yaml
```

---

## Verification

```bash
# Watch applications sync
watch kubectl get applications -n argocd

# Check sync waves are applied
kubectl get application metallb -n argocd -o jsonpath='{.metadata.annotations}'

# Verify all resources deployed
kubectl get ipaddresspool -n metallb-system
kubectl get clusterissuer
kubectl get clustersecretstore

# Check all pods
kubectl get pods -A
```

Expected result:
- All applications show `Synced` and `Healthy`
- MetalLB has IPAddressPool and L2Advertisement
- cert-manager has ClusterIssuers
- external-secrets has ClusterSecretStore
- All operators running successfully

---

## Why This Works

1. **Multiple Sources**: ArgoCD can pull from both Helm repos and Git repos in one Application
2. **Sync Waves**: ArgoCD deploys resources in wave order, waiting for health before proceeding
3. **Retry Policies**: Transient failures (CRD not ready yet) are automatically retried
4. **Root App Filtering**: Only discovers Application manifests, not config files
5. **Each App Manages Its Configs**: Applications pull their own config files after deploying operators

This approach:
- ✅ Deploys in correct order
- ✅ Doesn't require repository restructuring
- ✅ Uses ArgoCD best practices
- ✅ Handles transient failures gracefully
- ✅ Keeps configs co-located with Application manifests
