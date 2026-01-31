# K3s GitOps Homelab Stack (Flannel Edition)

## TL;DR

> **Quick Summary**: Set up a GitOps-managed single-node K3s homelab using default Flannel networking, ArgoCD, and full supporting infrastructure.
> 
> **Deliverables**:
> - K3s cluster with default Flannel CNI + Traefik ingress
> - ArgoCD managing all infrastructure via App-of-Apps
> - MetalLB for LoadBalancer IPs
> - Longhorn persistent storage
> - 1Password-backed secrets management (ESO)
> - TLS automation with Let's Encrypt
> - Full monitoring stack (Prometheus/Grafana)
> 
> **Estimated Effort**: Medium (1 day of focused work)
> **Parallel Execution**: YES - 2 waves after bootstrap
> **Critical Path**: ArgoCD Bootstrap → App-of-Apps → Everything Else

---

## Context

### Updated Approach
Using K3s defaults (Flannel + Traefik) instead of Cilium for simplicity:
- **CNI**: Flannel (K3s default) - no reinstallation needed
- **Ingress**: Traefik (K3s default) with IngressRoute CRDs
- **Load Balancer**: MetalLB L2 mode for LAN-accessible IPs
- **Secrets**: 1Password Connect + External Secrets Operator
- **TLS**: Let's Encrypt via Cloudflare DNS-01

### Key Settings
- **Network**: 10.200.x.x home network
- **LB Pool**: 10.200.100.0/27 (via MetalLB)
- **Domain**: k3s.s33g.uk (services at *.k3s.s33g.uk)
- **GitOps Repo**: This repo (infra/kube)

---

## Work Objectives

### Core Objective
Transform existing K3s into a GitOps-managed homelab with ArgoCD managing all infrastructure declaratively.

### Concrete Deliverables
1. GitOps repository structure in `infra/kube/`
2. ArgoCD managing all infrastructure via App-of-Apps
3. MetalLB providing LoadBalancer IPs
4. Working ingress at `*.k3s.s33g.uk` with automatic TLS
5. Persistent storage via Longhorn
6. Secrets from 1Password via ESO
7. Monitoring dashboards in Grafana

### Definition of Done
- [ ] `kubectl get applications -n argocd` shows all apps Synced
- [ ] `curl -k https://argocd.k3s.s33g.uk` returns ArgoCD UI
- [ ] `curl -k https://grafana.k3s.s33g.uk` returns Grafana UI
- [ ] Services get IPs from 10.200.100.x range
- [ ] No manual `kubectl apply` required after initial bootstrap

---

## Execution Strategy

```
Phase 1: GitOps Bootstrap (Sequential)
├── 1.1: Create GitOps repository structure
├── 1.2: Create bootstrap secrets documentation
├── 1.3: Install ArgoCD
└── 1.4: Apply root App-of-Apps

Phase 2: Infrastructure Wave (Parallel via ArgoCD)
├── 2.1: MetalLB (LoadBalancer IPs)
├── 2.2: Longhorn (Storage)
├── 2.3: cert-manager + ClusterIssuers
└── 2.4: External Secrets Operator

Phase 3: Services Wave (Parallel via ArgoCD)
├── 3.1: 1Password Connect
├── 3.2: ExternalDNS (Cloudflare)
├── 3.3: Traefik IngressRoutes
└── 3.4: kube-prometheus-stack

Phase 4: Validation
├── 4.1: Verify all apps synced
├── 4.2: Verify DNS + TLS
└── 4.3: Access dashboards
```

---

## TODOs

### Phase 1: GitOps Bootstrap

- [ ] 1.1. Create GitOps Repository Structure

  **What to do**:
  - Create directory structure for ArgoCD App-of-Apps pattern
  - Create all YAML files for infrastructure components
  
  **Directory Structure**:
  ```
  infra/kube/
  ├── bootstrap/
  │   ├── argocd/
  │   │   ├── namespace.yaml
  │   │   └── root-app.yaml
  │   └── secrets/
  │       └── README.md
  ├── infrastructure/
  │   ├── metallb/
  │   │   ├── application.yaml
  │   │   └── config.yaml
  │   ├── longhorn/
  │   │   └── application.yaml
  │   ├── cert-manager/
  │   │   ├── application.yaml
  │   │   └── issuers/
  │   │       ├── letsencrypt-staging.yaml
  │   │       └── letsencrypt-prod.yaml
  │   ├── external-secrets/
  │   │   ├── application.yaml
  │   │   └── secret-store.yaml
  │   ├── external-dns/
  │   │   └── application.yaml
  │   ├── 1password-connect/
  │   │   └── application.yaml
  │   └── monitoring/
  │       └── application.yaml
  ├── ingress/
  │   └── routes/
  │       ├── argocd.yaml
  │       ├── grafana.yaml
  │       └── longhorn.yaml
  └── apps/
      └── README.md
  ```

  **Acceptance Criteria**:
  ```bash
  find infra/kube -name "*.yaml" | wc -l
  # Expected: 15+ yaml files
  ```

  **Commit**: YES
  - Message: `feat(gitops): create ArgoCD app-of-apps repository structure`

---

- [ ] 1.2. Create Bootstrap Secrets Documentation

  **What to do**:
  - Document manual secrets (1Password creds, Cloudflare token)
  - These are created AFTER ArgoCD but BEFORE root app
  
  **Secrets Required**:
  1. `op-credentials` in `external-secrets` namespace
  2. `cloudflare-api-token` in `cert-manager` namespace
  3. `cloudflare-api-token` in `external-dns` namespace

  **Commit**: YES (with 1.1)

---

- [ ] 1.3. Install ArgoCD

  **Commands**:
  ```bash
  kubectl create namespace argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s
  
  # Get admin password
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
  ```

  **Acceptance Criteria**:
  ```bash
  kubectl get pods -n argocd
  # Expected: All pods Running
  ```

  **Commit**: NO (cluster operation)

---

- [ ] 1.4. Apply Root App-of-Apps

  **Commands**:
  ```bash
  kubectl apply -f infra/kube/bootstrap/argocd/root-app.yaml
  ```

  **Acceptance Criteria**:
  ```bash
  kubectl get applications -n argocd
  # Expected: 'root' application + child apps appearing
  ```

  **Commit**: NO (cluster operation)

---

### Phase 2: Infrastructure (ArgoCD-Managed)

- [ ] 2.1. MetalLB Configuration

  **Files** (infrastructure/metallb/):
  
  `application.yaml`:
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: metallb
    namespace: argocd
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
  ```

  `config.yaml`:
  ```yaml
  apiVersion: metallb.io/v1beta1
  kind: IPAddressPool
  metadata:
    name: homelab-pool
    namespace: metallb-system
  spec:
    addresses:
      - 10.200.100.1-10.200.100.30
  ---
  apiVersion: metallb.io/v1beta1
  kind: L2Advertisement
  metadata:
    name: homelab-l2
    namespace: metallb-system
  spec:
    ipAddressPools:
      - homelab-pool
  ```

  **Acceptance Criteria**:
  ```bash
  kubectl get ipaddresspool -n metallb-system
  # Expected: homelab-pool exists
  ```

---

- [ ] 2.2. Longhorn Storage

  **File** (infrastructure/longhorn/application.yaml):
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: longhorn
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: https://charts.longhorn.io
      chart: longhorn
      targetRevision: 1.7.2
      helm:
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
  ```

---

- [ ] 2.3. cert-manager with ClusterIssuers

  **Files** (infrastructure/cert-manager/):
  
  `application.yaml`:
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: cert-manager
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: https://charts.jetstack.io
      chart: cert-manager
      targetRevision: v1.16.2
      helm:
        values: |
          crds:
            enabled: true
    destination:
      server: https://kubernetes.default.svc
      namespace: cert-manager
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
      syncOptions:
        - CreateNamespace=true
  ```

  `issuers/letsencrypt-staging.yaml`:
  ```yaml
  apiVersion: cert-manager.io/v1
  kind: ClusterIssuer
  metadata:
    name: letsencrypt-staging
  spec:
    acme:
      server: https://acme-staging-v02.api.letsencrypt.org/directory
      email: your-email@s33g.uk
      privateKeySecretRef:
        name: letsencrypt-staging-key
      solvers:
        - dns01:
            cloudflare:
              apiTokenSecretRef:
                name: cloudflare-api-token
                key: api-token
          selector:
            dnsZones:
              - "s33g.uk"
  ```

---

- [ ] 2.4. External Secrets Operator

  **Files** (infrastructure/external-secrets/):
  
  `application.yaml`:
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: external-secrets
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: https://charts.external-secrets.io
      chart: external-secrets
      targetRevision: 0.12.1
      helm:
        values: |
          installCRDs: true
    destination:
      server: https://kubernetes.default.svc
      namespace: external-secrets
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
      syncOptions:
        - CreateNamespace=true
  ```

  `secret-store.yaml`:
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

---

### Phase 3: Services (ArgoCD-Managed)

- [ ] 3.1. 1Password Connect

  **File** (infrastructure/1password-connect/application.yaml):
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: onepassword-connect
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: https://1password.github.io/connect-helm-charts
      chart: connect
      targetRevision: 1.17.0
      helm:
        values: |
          connect:
            credentials_base64: ""
          existingSecret: op-credentials
    destination:
      server: https://kubernetes.default.svc
      namespace: external-secrets
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
  ```

---

- [ ] 3.2. ExternalDNS

  **File** (infrastructure/external-dns/application.yaml):
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: external-dns
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: https://kubernetes-sigs.github.io/external-dns
      chart: external-dns
      targetRevision: 1.15.0
      helm:
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
  ```

---

- [ ] 3.3. Traefik IngressRoutes

  **Files** (ingress/routes/):
  
  `argocd.yaml`:
  ```yaml
  apiVersion: traefik.io/v1alpha1
  kind: IngressRoute
  metadata:
    name: argocd
    namespace: argocd
    annotations:
      external-dns.alpha.kubernetes.io/hostname: argocd.k3s.s33g.uk
  spec:
    entryPoints:
      - websecure
    routes:
      - match: Host(`argocd.k3s.s33g.uk`)
        kind: Rule
        services:
          - name: argocd-server
            port: 80
    tls:
      secretName: argocd-tls
  ---
  apiVersion: cert-manager.io/v1
  kind: Certificate
  metadata:
    name: argocd-tls
    namespace: argocd
  spec:
    secretName: argocd-tls
    issuerRef:
      name: letsencrypt-staging
      kind: ClusterIssuer
    dnsNames:
      - argocd.k3s.s33g.uk
  ```

  `grafana.yaml`:
  ```yaml
  apiVersion: traefik.io/v1alpha1
  kind: IngressRoute
  metadata:
    name: grafana
    namespace: monitoring
    annotations:
      external-dns.alpha.kubernetes.io/hostname: grafana.k3s.s33g.uk
  spec:
    entryPoints:
      - websecure
    routes:
      - match: Host(`grafana.k3s.s33g.uk`)
        kind: Rule
        services:
          - name: kube-prometheus-stack-grafana
            port: 80
    tls:
      secretName: grafana-tls
  ---
  apiVersion: cert-manager.io/v1
  kind: Certificate
  metadata:
    name: grafana-tls
    namespace: monitoring
  spec:
    secretName: grafana-tls
    issuerRef:
      name: letsencrypt-staging
      kind: ClusterIssuer
    dnsNames:
      - grafana.k3s.s33g.uk
  ```

---

- [ ] 3.4. Monitoring Stack

  **File** (infrastructure/monitoring/application.yaml):
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: monitoring
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 67.4.0
      helm:
        values: |
          prometheus:
            prometheusSpec:
              retention: 7d
              storageSpec:
                volumeClaimTemplate:
                  spec:
                    storageClassName: longhorn
                    accessModes: ["ReadWriteOnce"]
                    resources:
                      requests:
                        storage: 10Gi
          grafana:
            persistence:
              enabled: true
              storageClassName: longhorn
              size: 5Gi
            adminPassword: "admin"
          alertmanager:
            alertmanagerSpec:
              storage:
                volumeClaimTemplate:
                  spec:
                    storageClassName: longhorn
                    accessModes: ["ReadWriteOnce"]
                    resources:
                      requests:
                        storage: 1Gi
    destination:
      server: https://kubernetes.default.svc
      namespace: monitoring
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
      syncOptions:
        - CreateNamespace=true
        - ServerSideApply=true
  ```

---

### Phase 4: Validation

- [ ] 4.1. Verify All ArgoCD Applications Synced

  ```bash
  kubectl get applications -n argocd -o custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status
  ```

- [ ] 4.2. Verify DNS + TLS

  ```bash
  dig argocd.k3s.s33g.uk +short
  curl -kI https://argocd.k3s.s33g.uk
  ```

- [ ] 4.3. Access Dashboards

  ```bash
  # ArgoCD: admin / (from argocd-initial-admin-secret)
  # Grafana: admin / admin
  curl -kI https://argocd.k3s.s33g.uk
  curl -kI https://grafana.k3s.s33g.uk
  ```

---

## Success Criteria

- [ ] All ArgoCD applications show Synced/Healthy
- [ ] Services get LoadBalancer IPs from MetalLB
- [ ] TLS certificates issued by Let's Encrypt
- [ ] DNS records automatically created in Cloudflare
- [ ] Grafana dashboards accessible
- [ ] No manual kubectl apply needed for day-to-day operations
