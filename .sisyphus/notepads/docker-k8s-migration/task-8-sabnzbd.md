# Task 8: SABnzbd Deployment - COMPLETED

## Summary
Successfully deployed SABnzbd (Usenet Downloader) to Kubernetes cluster.

## Files Created
1. **apps/media/sabnzbd/application.yaml** - ArgoCD Application
   - Sync wave: 3 (after secrets and PVCs)
   - Namespace: media
   - Auto-sync enabled with prune and self-heal

2. **apps/media/sabnzbd/deployment.yaml** - Deployment + Service
   - Image: ghcr.io/home-operations/sabnzbd:4.5.1
   - Port: 8080 (mapped to Service port 80)
   - Replicas: 1
   - Health checks: liveness (30s delay) and readiness (10s delay)
   - Resource limits: 1Gi memory, 1000m CPU
   - Volume mounts:
     - /config → sabnzbd-config-pvc (5Gi Longhorn)
     - /data → media-pvc (1Ti SMB)

3. **apps/media/sabnzbd/external-secret.yaml** - ExternalSecret
   - Syncs from 1Password: op://Infra/sabnzbd/
   - Fields: api_key, nzb_key
   - Refresh interval: 1h

4. **infrastructure/traefik/config/ingress-sabnzbd.yaml** - IngressRoute
   - Hostname: sabnzbd.k3s.s33g.uk
   - TLS: wildcard-k3s-s33g-uk-tls
   - MetalLB IP: 10.10.0.240
   - External DNS enabled

## Environment Variables
- TZ=Europe/London
- SABNZBD__API_KEY (from secret)
- SABNZBD__NZB_KEY (from secret)

## Dependencies Met
- ✓ Task 6 (Media PVCs) - sabnzbd-config-pvc and media-pvc already exist
- ✓ External Secrets Operator - onepassword ClusterSecretStore available
- ✓ Traefik - IngressRoute CRD available
- ✓ cert-manager - wildcard certificate available

## Validation
- ✓ All YAML files validated with kubectl dry-run
- ✓ Deployment includes proper health checks
- ✓ No hardcoded secrets
- ✓ Follows established patterns from cloudflared and prowlarr
- ✓ Committed with proper message format

## Commit
- Hash: 778384d
- Message: feat(media): deploy sabnzbd usenet downloader
- Files: 4 changed, 154 insertions

## Next Steps
- ArgoCD will auto-sync the application
- Monitor pod startup: `kubectl get pods -n media -l app.kubernetes.io/name=sabnzbd`
- Access web UI at: https://sabnzbd.k3s.s33g.uk
- Verify 1Password secrets are accessible
