# AGENTS.md - AI Coding Agent Guidelines

> K3s GitOps Homelab Infrastructure Repository

## Overview

Kubernetes infrastructure repo using **ArgoCD GitOps** for K3s homelab.

- **Stack**: K3s, ArgoCD, Traefik, MetalLB, Longhorn, cert-manager, External Secrets
- **Secrets**: 1Password SDK via External Secrets Operator
- **DNS/TLS**: Cloudflare + Let's Encrypt
- **Domain**: `*.k3s.s33g.uk`

## Directory Structure

```
├── apps/                   # Application workloads (authentik, gatus, uptime-kuma)
├── bootstrap/              # Initial setup (manual apply)
│   ├── argocd/            # Root app-of-apps
│   └── secrets/           # Bootstrap secrets (DO NOT COMMIT SECRETS)
├── infrastructure/         # Core platform components
│   ├── cert-manager/      # TLS certificates
│   ├── external-dns/      # Cloudflare DNS automation
│   ├── external-secrets/  # 1Password sync
│   ├── longhorn/          # Persistent storage
│   ├── metallb/           # LoadBalancer IPs
│   ├── monitoring/        # Prometheus + Grafana
│   └── traefik/           # Ingress + routes (config/ subdirectory)
└── .sisyphus/             # Planning docs
```

## Validation Commands

```bash
# Validate YAML manifest
kubectl apply --dry-run=client -f <file.yaml>

# Check ArgoCD apps
kubectl get applications -n argocd

# Force sync application
kubectl patch application <name> -n argocd --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# Check External Secrets
kubectl get externalsecret -A

# Check certificates
kubectl get certificate -A
```

## Code Style

### YAML Formatting
- **Indentation**: 2 spaces (no tabs)
- **Multi-doc**: Use `---` separator
- **Long strings**: Use `|` literal blocks for Helm values

### File Naming

| Type | Pattern |
|------|---------|
| ArgoCD App | `application.yaml` |
| External Secret | `external-secret.yaml` |
| IngressRoute | `ingress-<app>.yaml` |
| Middleware | `middleware-<name>.yaml` |
| Config | `config.yaml` |

### Resource Naming
- Names: `lowercase-hyphenated`
- Namespaces: `lowercase-hyphenated`
- Labels: Use `app.kubernetes.io/name`, `app.kubernetes.io/managed-by: argocd`

## ArgoCD Application Template

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"  # 0=CRDs, 1=secrets, 2+=apps
spec:
  project: default
  source:
    repoURL: <helm-repo>
    chart: <chart>
    targetRevision: <version>
    helm:
      values: |
        # inline values
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Traefik IngressRoute Template

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: <app>
  namespace: <namespace>
  annotations:
    external-dns.alpha.kubernetes.io/hostname: <app>.k3s.s33g.uk
    external-dns.alpha.kubernetes.io/target: 10.10.0.240
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`<app>.k3s.s33g.uk`)
      kind: Rule
      services:
        - name: <service>
          port: 80
  tls:
    secretName: wildcard-k3s-s33g-uk-tls
```

## External Secret Template

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: <name>
  namespace: <namespace>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: onepassword
    kind: ClusterSecretStore
  target:
    name: <k8s-secret-name>
  data:
    - secretKey: <key>
      remoteRef:
        key: op://Infra/<item>/<field>
```

## Important Conventions

1. **Sync Waves**: 0=CRDs/namespaces, 1=secrets/stores, 2+=apps
2. **Never commit secrets** - use External Secrets with 1Password
3. **TLS**: All routes use `wildcard-k3s-s33g-uk-tls`
4. **MetalLB IP**: Primary LoadBalancer at `10.10.0.240`

## Commit Format

```
<type>(<scope>): <description>
```

Types: `feat`, `fix`, `docs`, `chore`, `refactor`
Scopes: component name (traefik, authentik, external-secrets, etc.)

Examples:
```
feat(authentik): deploy SSO at sso.k3s.s33g.uk
fix(external-dns): correct boolean flag format
```

## Adding New App

1. Create `apps/<app>/application.yaml`
2. Add `external-secret.yaml` if secrets needed
3. Add `infrastructure/traefik/config/ingress-<app>.yaml`
4. Commit & push - ArgoCD auto-syncs

## Adding Infrastructure

1. Create `infrastructure/<component>/application.yaml`
2. Set appropriate sync-wave based on dependencies
3. Add config files as needed
