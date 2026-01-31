# Egress Gateway Deployment Guide

> **Infrastructure Context**: K3s v1.34.3 with Traefik v3.5.1, MetalLB, ArgoCD GitOps

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Patterns](#architecture-patterns)
3. [Pattern 1: Traefik + ExternalName (Recommended for Your Setup)](#pattern-1-traefik--externalname)
4. [Pattern 2: Network Policy Egress Control](#pattern-2-network-policy-egress-control)
5. [Pattern 3: Dedicated Egress Gateway with CNI](#pattern-3-dedicated-egress-gateway-with-cni)
6. [Deployment Steps](#deployment-steps)
7. [Testing & Validation](#testing--validation)
8. [Troubleshooting](#troubleshooting)

---

## Overview

### What is an Egress Gateway?

An **egress gateway** controls and secures outbound traffic from your Kubernetes cluster to external services. It provides:

- **Fixed egress IPs** for firewall whitelisting
- **Centralized traffic control** and policy enforcement
- **Observability** of outbound connections
- **Rate limiting** and security policies for external APIs

### When to Use Egress Gateways

| Use Case | Example |
|----------|---------|
| **External API Calls** | Payment gateways, third-party APIs |
| **Firewall Whitelisting** | Partner systems requiring fixed IPs |
| **Compliance** | Audit trails for external data access |
| **Rate Limiting** | Control API usage to external services |
| **Security** | Prevent direct pod-to-internet connections |

---

## Architecture Patterns

### Pattern Comparison

| Pattern | Complexity | Fixed Egress IP | Best For | Your Infrastructure |
|---------|------------|-----------------|----------|---------------------|
| **Traefik + ExternalName** | ⭐ Low | ❌ No (uses node IP) | Simple external API routing with policies | ✅ **Recommended** |
| **Network Policies** | ⭐ Low | ❌ No | Basic egress restrictions | ✅ Compatible |
| **Calico Egress Gateway** | ⭐⭐ Medium | ✅ Yes | Fixed IP requirements | ⚠️ Requires Calico CNI |
| **Cilium Egress Gateway** | ⭐⭐ Medium | ✅ Yes | Advanced eBPF features | ⚠️ Requires Cilium CNI |
| **Istio Egress Gateway** | ⭐⭐⭐ High | ✅ Yes | Full service mesh | ❌ Overkill for current setup |

---

## Pattern 1: Traefik + ExternalName

**✅ Recommended for your K3s + Traefik setup**

This pattern uses Traefik's existing infrastructure with `ExternalName` services to route traffic to external destinations with middleware policies.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Internal Pod (app)                                           │
│   └─> Calls: http://external-api.default.svc:443           │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ ExternalName Service (external-api)                         │
│   └─> Points to: api.example.com                           │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ Traefik Middleware (optional)                               │
│   ├─> Rate Limiting                                         │
│   ├─> Authentication                                        │
│   └─> Custom Headers                                        │
└─────────────────────────────────────────────────────────────┘
                           ↓
                  External Service
                  (api.example.com)
```

### Prerequisites

**1. Enable ExternalName Services in Traefik**

Create a HelmChart override (since K3s manages Traefik):

```bash
# Create override file
sudo mkdir -p /var/lib/rancher/k3s/server/manifests
sudo tee /var/lib/rancher/k3s/server/manifests/traefik-config.yaml <<EOF
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    deployment:
      podAnnotations:
        prometheus.io/port: "8082"
        prometheus.io/scrape: "true"
    providers:
      kubernetesIngress:
        publishedService:
          enabled: true
      kubernetesCRD:
        enabled: true
        allowExternalNameServices: true
        allowCrossNamespace: true
    priorityClassName: "system-cluster-critical"
    image:
      repository: "rancher/mirrored-library-traefik"
      tag: "3.5.1"
    tolerations:
    - key: "CriticalAddonsOnly"
      operator: "Exists"
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
    service:
      ipFamilyPolicy: "PreferDualStack"
EOF

# Traefik will automatically restart and apply changes
```

### Step-by-Step Implementation

#### 1. Create ExternalName Service

```yaml
# infrastructure/egress/external-api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-gateway-api
  namespace: default
  labels:
    app.kubernetes.io/name: payment-gateway
    app.kubernetes.io/component: egress
spec:
  type: ExternalName
  externalName: api.payment-provider.com  # External hostname
  ports:
    - name: https
      port: 443
      protocol: TCP
```

#### 2. (Optional) Create Middleware for Policies

```yaml
# infrastructure/traefik/config/middleware-external-api.yaml
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: external-api-ratelimit
  namespace: kube-system
spec:
  rateLimit:
    average: 100
    burst: 50
    period: 1m
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: external-api-headers
  namespace: kube-system
spec:
  headers:
    customRequestHeaders:
      X-Egress-Gateway: "traefik"
      X-Client-Id: "k3s-cluster"
    sslRedirect: true
    stsSeconds: 31536000
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: external-api-retry
  namespace: kube-system
spec:
  retry:
    attempts: 3
    initialInterval: 100ms
```

#### 3. (Optional) Create IngressRoute for Internal Access

If you want internal pods to access the external service through Traefik with policies:

```yaml
# infrastructure/traefik/config/ingress-external-api.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: external-api
  namespace: default
spec:
  entryPoints:
    - web
    - websecure
  routes:
    - match: Host(`external-api.internal`) && PathPrefix(`/api`)
      kind: Rule
      middlewares:
        - name: external-api-ratelimit
          namespace: kube-system
        - name: external-api-headers
          namespace: kube-system
        - name: external-api-retry
          namespace: kube-system
      services:
        - name: payment-gateway-api
          port: 443
  tls:
    secretName: wildcard-k3s-s33g-uk-tls
```

#### 4. Update Application to Use ExternalName Service

**Before (Direct External Call):**
```yaml
# Application calling external API directly
env:
  - name: PAYMENT_API_URL
    value: "https://api.payment-provider.com"
```

**After (Via ExternalName Service):**
```yaml
# Application calling via Kubernetes service
env:
  - name: PAYMENT_API_URL
    value: "https://payment-gateway-api.default.svc.cluster.local"
```

Or via internal IngressRoute:
```yaml
env:
  - name: PAYMENT_API_URL
    value: "https://external-api.internal/api"
```

---

## Pattern 2: Network Policy Egress Control

Use Kubernetes `NetworkPolicy` to restrict which pods can make external connections.

### Default Deny Egress Policy

```yaml
# infrastructure/network-policies/default-deny-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress: []  # Deny all egress by default
```

### Allow Specific External Destinations

```yaml
# infrastructure/network-policies/allow-payment-api.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-payment-api-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
    
    # Allow payment API (by IP)
    - to:
        - ipBlock:
            cidr: 203.0.113.0/24  # Payment API CIDR
      ports:
        - protocol: TCP
          port: 443
    
    # Allow internal services
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 80
        - protocol: TCP
          port: 443
```

### Allow Egress to Kubernetes API

```yaml
# infrastructure/network-policies/allow-k8s-api.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-k8s-api-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      needs-k8s-api: "true"
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              component: apiserver
      ports:
        - protocol: TCP
          port: 6443
```

---

## Pattern 3: Dedicated Egress Gateway with CNI

For fixed egress IPs, you need a CNI that supports egress gateway features.

### Option A: Install Calico Over K3s (Advanced)

**Warning**: This replaces K3s's default Flannel CNI.

```bash
# Disable Flannel in K3s
sudo systemctl stop k3s
sudo tee /etc/rancher/k3s/config.yaml <<EOF
flannel-backend: none
disable-network-policy: true
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
EOF
sudo systemctl start k3s

# Install Calico
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

**Calico Egress Gateway:**
```yaml
apiVersion: crd.projectcalico.org/v1
kind: EgressGateway
metadata:
  name: production-egress
  namespace: networking
spec:
  replicas: 2
  ipAddressPools:
    - cidr: 10.100.0.0/24
  nodeSelector:
    matchLabels:
      role: egress-gateway
---
apiVersion: crd.projectcalico.org/v1
kind: EgressGatewayPolicy
metadata:
  name: payment-api-egress
  namespace: production
spec:
  egressGateway: production-egress
  destinationCIDRs:
    - 203.0.113.0/24
  podSelector:
    matchLabels:
      app: payment-service
```

### Option B: Use NAT Gateway Node

Dedicate a node as a NAT gateway without changing CNI:

```yaml
# apps/egress-gateway/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: egress-gateway
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: egress-gateway
  template:
    metadata:
      labels:
        app: egress-gateway
    spec:
      hostNetwork: true
      nodeSelector:
        role: egress-gateway
      containers:
        - name: squid-proxy
          image: sameersbn/squid:3.5.27-2
          ports:
            - containerPort: 3128
          volumeMounts:
            - name: config
              mountPath: /etc/squid
      volumes:
        - name: config
          configMap:
            name: squid-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: squid-config
  namespace: kube-system
data:
  squid.conf: |
    http_port 3128
    acl localnet src 10.42.0.0/16
    http_access allow localnet
    http_access deny all
```

---

## Deployment Steps

### Using GitOps (Recommended for Your Setup)

#### 1. Create Directory Structure

```bash
mkdir -p infrastructure/egress
```

#### 2. Add ExternalName Services

```bash
# infrastructure/egress/external-services.yaml
cat > infrastructure/egress/external-services.yaml <<'EOF'
---
apiVersion: v1
kind: Service
metadata:
  name: stripe-api
  namespace: default
  labels:
    app.kubernetes.io/name: stripe
    app.kubernetes.io/component: egress
spec:
  type: ExternalName
  externalName: api.stripe.com
  ports:
    - name: https
      port: 443
      protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: sendgrid-api
  namespace: default
  labels:
    app.kubernetes.io/name: sendgrid
    app.kubernetes.io/component: egress
spec:
  type: ExternalName
  externalName: api.sendgrid.com
  ports:
    - name: https
      port: 443
      protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: github-api
  namespace: default
  labels:
    app.kubernetes.io/name: github
    app.kubernetes.io/component: egress
spec:
  type: ExternalName
  externalName: api.github.com
  ports:
    - name: https
      port: 443
      protocol: TCP
EOF
```

#### 3. Add Middleware Policies

```bash
# infrastructure/traefik/config/middleware-egress.yaml
cat > infrastructure/traefik/config/middleware-egress.yaml <<'EOF'
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: egress-ratelimit-high
  namespace: kube-system
spec:
  rateLimit:
    average: 1000
    burst: 500
    period: 1m
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: egress-ratelimit-medium
  namespace: kube-system
spec:
  rateLimit:
    average: 100
    burst: 50
    period: 1m
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: egress-ratelimit-low
  namespace: kube-system
spec:
  rateLimit:
    average: 10
    burst: 5
    period: 1m
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: egress-headers
  namespace: kube-system
spec:
  headers:
    customRequestHeaders:
      X-Egress-Gateway: "k3s-traefik"
      X-Forwarded-Proto: "https"
    sslRedirect: true
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: egress-retry
  namespace: kube-system
spec:
  retry:
    attempts: 3
    initialInterval: 100ms
EOF
```

#### 4. Create ArgoCD Application

```bash
# infrastructure/egress/application.yaml
cat > infrastructure/egress/application.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: egress-config
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  source:
    repoURL: git@github.com:s33g/infra.git
    targetRevision: HEAD
    path: kube/infrastructure/egress
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

#### 5. Commit and Push

```bash
git add infrastructure/egress/ infrastructure/traefik/config/middleware-egress.yaml
git commit -m "feat(egress): add egress gateway configuration

- Add ExternalName services for external APIs
- Add Traefik middleware for rate limiting and headers
- Add ArgoCD application for egress config management"
git push
```

#### 6. Verify Deployment

```bash
# Check ArgoCD sync
kubectl get application egress-config -n argocd

# Check services
kubectl get svc -l app.kubernetes.io/component=egress

# Check middleware
kubectl get middleware -n kube-system | grep egress
```

---

## Testing & Validation

### Test 1: Verify ExternalName Resolution

```bash
# Create test pod
kubectl run test-egress --rm -it --image=nicolaka/netshoot -- /bin/bash

# Inside pod:
# Test DNS resolution
nslookup stripe-api.default.svc.cluster.local

# Expected output:
# Server:    10.43.0.10
# Address:   10.43.0.10#53
# stripe-api.default.svc.cluster.local canonical name = api.stripe.com.

# Test connection
curl -I https://stripe-api.default.svc.cluster.local:443
```

### Test 2: Verify Rate Limiting

```bash
# Apply test IngressRoute with rate limit
kubectl apply -f - <<EOF
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: test-egress-ratelimit
  namespace: default
spec:
  entryPoints:
    - web
  routes:
    - match: Host(\`test-egress.internal\`)
      kind: Rule
      middlewares:
        - name: egress-ratelimit-low
          namespace: kube-system
      services:
        - name: stripe-api
          port: 443
EOF

# Hammer the endpoint
for i in {1..20}; do
  curl -H "Host: test-egress.internal" http://10.10.0.240 -I
done

# Should see 429 Too Many Requests after 10 requests
```

### Test 3: Verify Network Policies

```bash
# Create test namespace with deny-all egress
kubectl create namespace test-egress
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: test-egress
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress: []
EOF

# Create pod in test namespace
kubectl run test-blocked -n test-egress --image=nicolaka/netshoot -- sleep 3600

# Try external connection (should fail)
kubectl exec -n test-egress test-blocked -- curl -I https://google.com --max-time 5
# Expected: timeout or connection refused

# Cleanup
kubectl delete namespace test-egress
```

---

## Troubleshooting

### Issue 1: ExternalName Service Not Resolving

**Symptom**: `nslookup` fails or returns NXDOMAIN

**Solutions**:
```bash
# Check CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check service exists
kubectl get svc stripe-api -o yaml

# Verify externalName field
kubectl get svc stripe-api -o jsonpath='{.spec.externalName}'

# Test DNS from pod
kubectl run test-dns --rm -it --image=busybox -- nslookup stripe-api.default.svc.cluster.local
```

### Issue 2: Traefik Not Routing to ExternalName

**Symptom**: 404 or 502 errors when accessing via IngressRoute

**Solutions**:
```bash
# Check if allowExternalNameServices is enabled
kubectl get helmchart traefik -n kube-system -o yaml | grep allowExternalNameServices

# If not enabled, update HelmChartConfig
sudo tee /var/lib/rancher/k3s/server/manifests/traefik-config.yaml <<EOF
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    providers:
      kubernetesCRD:
        allowExternalNameServices: true
EOF

# Check Traefik logs
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik --tail=100
```

### Issue 3: Rate Limiting Not Working

**Symptom**: No 429 errors even after exceeding limits

**Solutions**:
```bash
# Verify middleware exists
kubectl get middleware egress-ratelimit-low -n kube-system -o yaml

# Check IngressRoute references middleware correctly
kubectl get ingressroute <name> -o yaml | grep -A 5 middlewares

# Enable Traefik debug logging
kubectl edit helmchart traefik -n kube-system
# Add: log.level: DEBUG

# Check Traefik logs for rate limit events
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik | grep -i rate
```

### Issue 4: Network Policy Blocking Legitimate Traffic

**Symptom**: Pods can't connect to expected services

**Solutions**:
```bash
# List all network policies
kubectl get networkpolicy -A

# Describe specific policy
kubectl describe networkpolicy deny-all-egress -n production

# Temporarily disable to test
kubectl delete networkpolicy deny-all-egress -n production

# Check pod logs for connection errors
kubectl logs <pod-name> -n production

# Add allow rule for DNS
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
EOF
```

### Debug Commands

```bash
# Check external connectivity from node
ssh deathstar
curl -I https://api.stripe.com

# Check from pod
kubectl run test-net --rm -it --image=nicolaka/netshoot -- curl -I https://api.stripe.com

# Trace DNS resolution
kubectl run test-net --rm -it --image=nicolaka/netshoot -- dig stripe-api.default.svc.cluster.local

# Check Traefik routing
kubectl get ingressroute -A
kubectl describe ingressroute <name> -n <namespace>

# View Traefik dashboard
kubectl port-forward -n kube-system svc/traefik 9000:9000
# Access: http://localhost:9000/dashboard/
```

---

## Best Practices

### 1. Use Descriptive Service Names

```yaml
# Good
name: stripe-payment-api
name: sendgrid-email-api

# Bad
name: external-api-1
name: api-service
```

### 2. Add Labels for Organization

```yaml
metadata:
  labels:
    app.kubernetes.io/name: stripe
    app.kubernetes.io/component: egress
    app.kubernetes.io/managed-by: argocd
    environment: production
    cost-center: payments
```

### 3. Document External Dependencies

```yaml
# infrastructure/egress/external-services.yaml
---
# Stripe Payment API
# Used by: payment-service, subscription-service
# Rate Limit: 100 req/sec
# Docs: https://stripe.com/docs/api
apiVersion: v1
kind: Service
metadata:
  name: stripe-api
  annotations:
    description: "Stripe Payment Processing API"
    rate-limit: "100/sec"
    documentation: "https://stripe.com/docs/api"
spec:
  type: ExternalName
  externalName: api.stripe.com
```

### 4. Use Different Rate Limits Per Service

```yaml
# High-volume internal tools
middlewares:
  - name: egress-ratelimit-high  # 1000/min

# Production APIs
middlewares:
  - name: egress-ratelimit-medium  # 100/min

# Development/testing
middlewares:
  - name: egress-ratelimit-low  # 10/min
```

### 5. Monitor Egress Traffic

```yaml
# Add Prometheus annotations
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: stripe-api
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "/metrics"
spec:
  # ... routes
```

---

## Summary

### Quick Start for Your Infrastructure

1. **Enable ExternalName** in Traefik (via HelmChartConfig)
2. **Create ExternalName services** for each external API
3. **Add middleware** for rate limiting and headers
4. **Update applications** to use service names instead of direct URLs
5. **Add NetworkPolicies** for additional security (optional)
6. **Monitor and iterate** based on actual usage

### Recommended Pattern

✅ **Start with**: Traefik + ExternalName + Middleware
- Low complexity
- No infrastructure changes
- Reuses existing Traefik
- GitOps-friendly

⚠️ **Consider later**: Calico/Cilium egress gateway
- Only if you need fixed egress IPs
- Requires CNI migration
- More operational complexity

### Next Steps

1. Choose external APIs to route through egress
2. Create ExternalName services
3. Add middleware policies
4. Update application configurations
5. Test and validate
6. Add to ArgoCD for GitOps management

---

**Questions or Issues?**
- Check Traefik logs: `kubectl logs -n kube-system -l app.kubernetes.io/name=traefik`
- Review this guide's [Troubleshooting](#troubleshooting) section
- Check Traefik docs: https://doc.traefik.io/traefik/
