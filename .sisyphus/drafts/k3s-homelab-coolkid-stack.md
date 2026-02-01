# Draft: K3s "Cool Kid" 2026 Homelab Stack

## Requirements (confirmed)

### Secrets Backend
- **Choice**: 1Password Connect
- Self-hosted 1Password Connect server required
- ESO will integrate with 1Password

### Network Configuration
- **IP Range**: 10.200.x.x (user's home network)
- **LB Pool**: 10.200.100.0/27 (32 IPs: 10.200.100.1-30 usable)
- **Pod CIDR**: 10.42.0.0/16 (K3s default)

### Monitoring
- **Choice**: Include in Phase 1
- Full Prometheus/Grafana stack from day 1
- Essential for debugging and visibility

### Node Information
- **Hostname**: deathstar.home
- **Setup**: Single node K3s
- **State**: Fresh start, no existing workloads

## Technical Decisions

### CNI: Cilium (Research Complete)
- Replace Flannel with Cilium (eBPF-based)
- Use Cilium native LoadBalancer (NOT MetalLB) - recommended for new deployments in 2026
- Enable Hubble for observability
- K3s flags needed:
  - `--flannel-backend=none`
  - `--disable-network-policy`
  - `--disable=servicelb`
  - `--disable-kube-proxy` (for kube-proxy-free mode)
- Pod CIDR: 10.42.0.0/16 (K3s default)

### Gateway API (Research Complete)
- Gateway API v1.4 is GA (Oct 2025) - production ready
- **Recommendation**: Use Cilium Gateway API (native integration)
- No need for separate ingress controller
- nginx-ingress is being archived - avoid
- cert-manager supports Gateway API via annotations
- ExternalDNS supports Gateway API sources

### Ingress Strategy
- **Choice**: Cilium Gateway API (CONFIRMED)
- Native eBPF integration, zero extra deployment
- Gateway API v1.4 conformant

### TLS Strategy
- **Choice**: Let's Encrypt Staging (start safe, upgrade to prod later)
- cert-manager with ACME
- DNS-01 challenge via Cloudflare API
- Domain: k3s.s33g.uk (wildcard cert for *.k3s.s33g.uk)

### DNS Configuration
- **Provider**: Cloudflare
- **Domain**: k3s.s33g.uk
- **Pattern**: Services at *.k3s.s33g.uk (e.g., argocd.k3s.s33g.uk, grafana.k3s.s33g.uk)
- **ExternalDNS**: Will auto-create A records pointing to LoadBalancer IPs
- **Cloudflare API Token**: Required for cert-manager and ExternalDNS (store in 1Password)

### GitOps Repository
- **Choice**: This repo (infra/kube)
- Add GitOps structure to current repository
- ArgoCD will sync from this location

### 1Password Connect
- **Choice**: Deploy in-cluster
- 1Password Connect needs to be set up as part of this plan
- Account type: Family/Individual
- Will be managed by ArgoCD after bootstrap

### GitOps Structure
- **Choice**: Single environment (simple monorepo)
- Structure: apps/, infrastructure/, bootstrap/
- Perfect for single-cluster homelab

### Monitoring Stack
- **Choice**: kube-prometheus-stack (Helm)
- Includes: Prometheus, Grafana, AlertManager, ServiceMonitors
- Pre-built Kubernetes dashboards included

### Renovate
- **Choice**: Skip for now
- Can add later via GitHub App

## Research Findings

### Cilium + K3s Best Practices (2024-2026)
Source: Cilium official docs, community guides

1. **Installation**: Use Cilium CLI for most cases, Helm for GitOps workflows
2. **Critical**: Must set `ipam.operator.clusterPoolIPv4PodCIDRList="10.42.0.0/16"`
3. **Hubble**: Enable with `cilium hubble enable --ui`
4. **LoadBalancer**: Use CiliumLoadBalancerIPPool CRD for IP allocation
5. **Single-node gotchas**:
   - CRD initialization timing issues possible
   - Minimum 2GB RAM (4GB recommended with Hubble)
   - Kernel >= 5.10 required

### Gateway API Findings
- v1.4 GA - stable and production-ready
- Cilium passes all Core conformance tests
- BackendTLSPolicy for mTLS to services
- ExternalDNS auto-creates DNS from HTTPRoute hostnames

## Open Questions

1. ~~**LoadBalancer IP Range**~~: ANSWERED - 10.200.100.0/27
2. ~~**Gateway API vs Traefik**~~: ANSWERED - Cilium Gateway API
3. ~~**GitHub Repository**~~: ANSWERED - This repo (infra/kube)
4. **DNS**: Using external DNS provider or local (Pi-hole, etc.)? STILL OPEN
5. ~~**TLS Strategy**~~: ANSWERED - Let's Encrypt Staging
6. ~~**1Password Setup**~~: ANSWERED - Deploy in-cluster

### All Questions Answered
- ~~DNS provider~~: Cloudflare (excellent for Let's Encrypt DNS-01)
- ~~1Password account~~: Family/Individual
- ~~Domain name~~: k3s.s33g.uk (services at *.k3s.s33g.uk)
- ~~Renovate~~: Skip for now, add later

## Scope Boundaries

### INCLUDE
- K3s reinstallation with Cilium-ready flags
- Cilium CNI + Hubble
- Cilium native LoadBalancer
- Cilium Gateway API
- ArgoCD (App-of-Apps pattern)
- Longhorn CSI storage
- External Secrets Operator + 1Password
- cert-manager
- Monitoring stack (Prometheus/Grafana)
- GitOps repository structure

### EXCLUDE (Guardrails)
- Multi-node HA setup (single node focus)
- Service mesh features beyond basic Gateway API
- Complex BGP peering (using L2 announcements)
- Production TLS (unless user wants Let's Encrypt)
- Backup infrastructure (S3/MinIO) - configure Longhorn for it but don't deploy

## Bootstrap Sequence (Draft)

### Phase 0: Node Preparation
1. Reinstall K3s with Cilium-ready flags
2. Install Cilium CLI
3. Install Cilium + Hubble

### Phase 1: Manual Bootstrap (ONE-TIME kubectl apply)
1. Install ArgoCD namespace + CRDs
2. Install ArgoCD itself
3. Apply root Application (App-of-Apps)

### Phase 2: ArgoCD-Managed (Everything else)
- ArgoCD manages itself
- Longhorn
- External Secrets Operator
- cert-manager
- Monitoring stack
- Cilium LoadBalancer IP Pool
- Gateway resources

## Test Strategy Decision
- **Infrastructure exists**: NO (fresh repo)
- **Testing approach**: Automated verification via kubectl/curl commands
- **QA approach**: Agent-executable verification:
  - kubectl commands to verify deployments, pods, services
  - curl commands to test endpoints
  - Cilium connectivity tests
  - ArgoCD sync status checks
  - Gateway API route validation

## Verification Strategy

Each phase will be verified with automated commands:
- Phase 0: `cilium status --wait`, `cilium connectivity test`
- Phase 1: `kubectl get applications -n argocd`, ArgoCD UI accessible
- Phase 2: Service endpoints responding, TLS working, secrets resolving

## Final Decisions Summary

| Component | Choice |
|-----------|--------|
| CNI | Cilium (eBPF, kube-proxy replacement) |
| LoadBalancer | Cilium native LB, 10.200.100.0/27 |
| Ingress | Cilium Gateway API |
| GitOps | ArgoCD, App-of-Apps pattern |
| Storage | Longhorn CSI |
| Secrets | ESO + 1Password Connect (in-cluster) |
| TLS | cert-manager + Let's Encrypt Staging (DNS-01 via Cloudflare) |
| DNS | Cloudflare + ExternalDNS |
| Domain | k3s.s33g.uk (*.k3s.s33g.uk) |
| Monitoring | kube-prometheus-stack |
| Repo | This repo (infra/kube), single env structure |
