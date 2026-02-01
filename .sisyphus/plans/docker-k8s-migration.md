# Docker to Kubernetes Migration Plan

## TL;DR

> **Quick Summary**: Migrate Docker containers from c3po.home to K3s cluster running on the same machine, using SMB CSI driver for media storage access. Conservative, phased approach prioritizing quick wins.
> 
> **Deliverables**:
> - SMB CSI driver installation with NAS storage access
> - Cloudflared tunnel migration to K8s
> - Monitoring exporters migration (speedtest-exporter)
> - Syslog-ng deployment
> - Media stack migration (Plex, Sonarr, Radarr, Prowlarr, SABnzbd, etc.)
> - Go API + Redis deployment (later phase)
> - Decommissioning of redundant Docker containers
> 
> **Estimated Effort**: Large (multi-phase, 2-3 weeks)
> **Parallel Execution**: YES - 3 waves
> **Critical Path**: SMB CSI -> Media Storage PVs -> Media Apps

---

## Context

### Original Request
Migrate Docker containers running on c3po.home to the existing K3s cluster, which also runs on c3po.home.

### Container Inventory (from docker ps)

| Container | Image | Ports | Decision |
|-----------|-------|-------|----------|
| go-api | public-api | 8069 | MIGRATE (Phase 4) |
| go-redis | redis:alpine | 6379 | MIGRATE (Phase 4) |
| sabnzbd | ghcr.io/home-operations/sabnzbd:4.5.1 | 6904 | MIGRATE (Phase 3) |
| sonarr | lscr.io/linuxserver/sonarr:4.0.14 | 6902 | MIGRATE (Phase 3) |
| netbootxyz | ghcr.io/netbootxyz/netbootxyz | 69/udp, 3000, 8080 | KEEP ON DOCKER |
| grafana | grafana/grafana | 3060 | DECOMMISSION (use K8s) |
| speedtest-exporter | miguelndecarvalho/speedtest-exporter | 9798 | MIGRATE (Phase 2) |
| node_exporter | quay.io/prometheus/node-exporter | 9100 | KEEP ON DOCKER (host-specific) |
| syslog-ng | lscr.io/linuxserver/syslog-ng | 514/udp, 601, 6514 | MIGRATE (Phase 2) |
| prometheus | prom/prometheus | 9090 | DECOMMISSION (use K8s) |
| nginx_preseed | nginx:alpine | 8885 | KEEP ON DOCKER (with netbootxyz) |
| fivem | spritsail/fivem | 3069, 30120, etc. | KEEP ON DOCKER |
| nginx-internal | nginx:1.27.3 | - | DECOMMISSION |
| smartctl-exporter | prometheuscommunity/smartctl-exporter | 9633 | KEEP ON DOCKER (host-specific) |
| radarr | lscr.io/linuxserver/radarr:5.23.3 | 6903 | MIGRATE (Phase 3) |
| prowlarr | lscr.io/linuxserver/prowlarr:1.35.1 | 6901 | MIGRATE (Phase 3) |
| unpackerr | ghcr.io/unpackerr/unpackerr:0.14.5 | - | MIGRATE (Phase 3) |
| plex | lscr.io/linuxserver/plex:1.41.7 | 6900 | MIGRATE (Phase 3) |
| metube | ghcr.io/alexta69/metube | 8081 | MIGRATE (Phase 3) |
| pinchflat | ghcr.io/kieraneglin/pinchflat | 8945 | MIGRATE (Phase 3) |
| deemix | ghcr.io/bambanah/deemix:v4.3.3 | 6595 | MIGRATE (Phase 3) |
| nginx | nginx:1.27.3 | - | DECOMMISSION (broken) |
| cloudflared | cloudflare/cloudflared | - | MIGRATE (Phase 1) |
| uptime | louislam/uptime-kuma | - | DECOMMISSION (already in K8s) |
| cloudflare-ddns | favonia/cloudflare-ddns | - | KEEP ON DOCKER |

### Interview Summary

**Key Decisions**:
- **Storage**: Use SMB CSI driver from start (proper architecture)
- **Monitoring**: Migrate to K8s kube-prometheus-stack
- **Game/PXE**: Keep FiveM and NetbootXYZ on Docker (special networking)
- **Strategy**: Conservative - quick wins first
- **Media Stack**: Migrate all Arr apps together as a unit
- **Nginx containers**: Decommission (broken/redundant with Traefik)

### Existing K8s Infrastructure

Already deployed in K8s:
- ArgoCD (GitOps)
- Traefik (ingress with IngressRoute CRDs)
- MetalLB (LoadBalancer at 10.10.0.240)
- Longhorn (storage)
- cert-manager (TLS with Let's Encrypt)
- External Secrets (1Password integration)
- External DNS (Cloudflare)
- kube-prometheus-stack (Prometheus + Grafana)
- Authentik (SSO)
- Gatus (status page)
- Uptime Kuma

---

## Work Objectives

### Core Objective
Systematically migrate Docker containers to Kubernetes, leveraging existing GitOps (ArgoCD) patterns, while maintaining service continuity.

### Concrete Deliverables
- `infrastructure/smb-csi/application.yaml` - SMB CSI driver installation
- `infrastructure/smb-csi/storageclass.yaml` - StorageClass for NAS
- `infrastructure/smb-csi/external-secret.yaml` - SMB credentials
- `apps/cloudflared/application.yaml` - Cloudflared tunnel
- `apps/syslog-ng/application.yaml` - Syslog-ng deployment
- `apps/speedtest-exporter/application.yaml` - Speedtest exporter
- `apps/media/` - Media stack (Plex, *arr apps)
- `apps/go-api/` - Go API + Redis (Phase 4)

### Definition of Done
- [ ] All migrated services accessible via `*.k3s.s33g.uk`
- [ ] Media stack can read/write to NAS storage
- [ ] Docker containers for migrated services stopped
- [ ] ArgoCD shows all applications synced and healthy

### Must Have
- SMB storage access for media applications
- TLS termination via existing wildcard certificate
- External Secrets integration for credentials
- Health checks and readiness probes

### Must NOT Have (Guardrails)
- DO NOT migrate FiveM, NetbootXYZ, or nginx_preseed
- DO NOT migrate node_exporter or smartctl-exporter (host-specific)
- DO NOT use hostPath volumes (use SMB CSI properly)
- DO NOT create services without IngressRoute for web UIs
- DO NOT hardcode secrets in YAML (use External Secrets)
- DO NOT remove Docker containers until K8s version verified working

---

## Verification Strategy

### Test Decision
- **Infrastructure exists**: YES (K8s cluster operational)
- **User wants tests**: Manual verification
- **QA approach**: Automated verification via kubectl + curl

### Automated Verification (per phase)

Each phase will be verified by:
1. `kubectl get pods -n <namespace>` - All pods Running
2. `kubectl get application -n argocd` - App Synced/Healthy
3. `curl -k https://<app>.k3s.s33g.uk` - HTTP 200 response
4. Application-specific health checks

---

## Execution Strategy

### Parallel Execution Waves

```
Phase 1 - Infrastructure (Wave 1):
├── Task 1: SMB CSI Driver
└── Task 2: Cloudflared Tunnel

Phase 2 - Quick Wins (Wave 2, after Phase 1):
├── Task 3: Speedtest Exporter
├── Task 4: Syslog-ng
└── Task 5: Decommission redundant containers

Phase 3 - Media Stack (Wave 3, after SMB CSI ready):
├── Task 6: Media Storage PVCs
├── Task 7: Prowlarr (indexer manager)
├── Task 8: SABnzbd (downloader)
├── Task 9: Sonarr (TV)
├── Task 10: Radarr (Movies)
├── Task 11: Plex (streaming)
├── Task 12: Supporting apps (Unpackerr, Deemix, Metube, Pinchflat)

Phase 4 - Lower Priority (Wave 4):
├── Task 13: Go API + Redis
└── Task 14: Final cleanup

Critical Path: Task 1 → Task 6 → Tasks 7-12
```

### Dependency Matrix

| Task | Depends On | Blocks | Parallel With |
|------|------------|--------|---------------|
| 1 (SMB CSI) | None | 6 | 2 |
| 2 (Cloudflared) | None | None | 1 |
| 3 (Speedtest) | None | None | 4, 5 |
| 4 (Syslog-ng) | None | None | 3, 5 |
| 5 (Decommission) | None | None | 3, 4 |
| 6 (Media PVCs) | 1 | 7-12 | None |
| 7-12 (Media apps) | 6 | None | Each other |
| 13 (Go API) | None | None | 14 |
| 14 (Cleanup) | 7-12 | None | 13 |

---

## TODOs

### Phase 1: Infrastructure Foundation

- [ ] 1. Install SMB CSI Driver

  **What to do**:
  - Create ArgoCD Application for SMB CSI driver Helm chart
  - Create ExternalSecret for SMB credentials (username/password from 1Password)
  - Create StorageClass for NAS media share
  - Test with a sample PVC

  **Must NOT do**:
  - Do not use anonymous auth
  - Do not hardcode credentials

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Infrastructure setup with known patterns
  - **Skills**: [`git-master`]
    - `git-master`: For proper commits

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Task 2)
  - **Blocks**: Task 6 (Media PVCs)
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `infrastructure/external-secrets/application.yaml` - ArgoCD app pattern with Helm
  - `infrastructure/external-secrets/secret-store.yaml` - ExternalSecret to ClusterSecretStore

  **API/Type References**:
  - SMB CSI Helm chart: `https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/charts`
  - Chart version: v1.18.0

  **External References**:
  - Official docs: https://github.com/kubernetes-csi/csi-driver-smb
  - StorageClass example: https://github.com/kubernetes-csi/csi-driver-smb/blob/master/deploy/example/storageclass-smb.yaml

  **Acceptance Criteria**:

  ```bash
  # Verify CSI driver pods running
  kubectl get pods -n kube-system -l app.kubernetes.io/name=csi-driver-smb
  # Expected: 2+ pods Running (controller + node)

  # Verify StorageClass created
  kubectl get storageclass smb-media
  # Expected: smb-media provisioner: smb.csi.k8s.io

  # Verify secret created from 1Password
  kubectl get secret smb-credentials -n kube-system
  # Expected: Secret exists with username/password keys
  ```

  **Files to create**:
  - `infrastructure/smb-csi/application.yaml`
  - `infrastructure/smb-csi/external-secret.yaml`
  - `infrastructure/smb-csi/storageclass.yaml`

  **Commit**: YES
  - Message: `feat(smb-csi): install SMB CSI driver for NAS storage access`
  - Files: `infrastructure/smb-csi/*`

---

- [ ] 2. Migrate Cloudflared Tunnel to Kubernetes

  **What to do**:
  - Create ArgoCD Application for cloudflared
  - Store TUNNEL_TOKEN in 1Password and reference via ExternalSecret
  - Create Deployment with cloudflared running managed tunnel
  - Optionally create IngressRoute for tunnel dashboard (if available)

  **Must NOT do**:
  - Do not expose tunnel token in Git
  - Do not modify existing tunnel config in Cloudflare dashboard yet

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Stateless deployment, straightforward
  - **Skills**: [`git-master`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Task 1)
  - **Blocks**: None
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `apps/uptime-kuma/application.yaml` - Simple app deployment pattern
  - `apps/authentik/application.yaml` - App with ExternalSecret reference

  **External References**:
  - Cloudflare docs: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/deploy-tunnels/deployment-guides/kubernetes/
  - Community Helm chart: https://github.com/cloudflare/helm-charts

  **Acceptance Criteria**:

  ```bash
  # Verify cloudflared pod running
  kubectl get pods -n cloudflared -l app=cloudflared
  # Expected: 1 pod Running

  # Verify tunnel is connected (check logs)
  kubectl logs -n cloudflared -l app=cloudflared --tail=20 | grep -i "connection"
  # Expected: "Connection registered" messages

  # Verify external access still works (via tunnel)
  curl -s https://your-tunneled-domain.com | head -5
  # Expected: Valid response from tunneled service
  ```

  **Files to create**:
  - `apps/cloudflared/application.yaml`
  - `apps/cloudflared/external-secret.yaml`
  - `apps/cloudflared/deployment.yaml` (or inline Helm values)

  **Commit**: YES
  - Message: `feat(cloudflared): migrate cloudflare tunnel to kubernetes`
  - Files: `apps/cloudflared/*`

---

### Phase 2: Quick Wins

- [ ] 3. Deploy Speedtest Exporter

  **What to do**:
  - Create ArgoCD Application for speedtest-exporter
  - Configure Prometheus scrape (add ServiceMonitor)
  - Verify metrics visible in Grafana

  **Must NOT do**:
  - Do not schedule speedtests too frequently (bandwidth consumption)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Simple stateless exporter
  - **Skills**: [`git-master`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 4, 5)
  - **Blocks**: None
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `infrastructure/monitoring/application.yaml` - ServiceMonitor pattern with kube-prometheus-stack

  **External References**:
  - Docker image: `miguelndecarvalho/speedtest-exporter`
  - Grafana dashboard: https://grafana.com/grafana/dashboards/13665

  **Acceptance Criteria**:

  ```bash
  # Verify pod running
  kubectl get pods -n monitoring -l app=speedtest-exporter
  # Expected: 1 pod Running

  # Verify metrics endpoint
  kubectl port-forward -n monitoring svc/speedtest-exporter 9798:9798 &
  curl -s localhost:9798/metrics | grep speedtest
  # Expected: speedtest_* metrics present

  # Verify ServiceMonitor
  kubectl get servicemonitor -n monitoring speedtest-exporter
  # Expected: ServiceMonitor exists
  ```

  **Files to create**:
  - `apps/speedtest-exporter/application.yaml` (with inline manifests or Helm)

  **Commit**: YES
  - Message: `feat(monitoring): add speedtest-exporter to kubernetes`
  - Files: `apps/speedtest-exporter/*`

---

- [ ] 4. Deploy Syslog-ng

  **What to do**:
  - Create ArgoCD Application for syslog-ng
  - Use wiremind/syslog-ng Helm chart or linuxserver image
  - Configure to receive syslog on UDP 514, TCP 601, TCP 6514
  - Mount PVC for log storage (use Longhorn)
  - Create LoadBalancer service for external syslog reception

  **Must NOT do**:
  - Do not expose syslog ports to internet (internal only)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Standard deployment with persistence
  - **Skills**: [`git-master`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 3, 5)
  - **Blocks**: None
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `apps/uptime-kuma/application.yaml` - App with Longhorn PVC

  **External References**:
  - Helm chart: https://artifacthub.io/packages/helm/wiremind/syslog-ng
  - Image: linuxserver/syslog-ng

  **Acceptance Criteria**:

  ```bash
  # Verify pod running
  kubectl get pods -n syslog-ng
  # Expected: 1 pod Running

  # Verify service with LoadBalancer IP
  kubectl get svc -n syslog-ng
  # Expected: LoadBalancer with external IP

  # Test syslog reception (from another host)
  logger -n <syslog-lb-ip> -P 514 "Test message from K8s migration"
  kubectl logs -n syslog-ng -l app=syslog-ng --tail=5
  # Expected: Test message appears in logs
  ```

  **Files to create**:
  - `apps/syslog-ng/application.yaml`

  **Commit**: YES
  - Message: `feat(syslog-ng): deploy centralized syslog to kubernetes`
  - Files: `apps/syslog-ng/*`

---

- [ ] 5. Decommission Redundant Docker Containers

  **What to do**:
  - Stop and remove nginx container (broken)
  - Stop and remove nginx-internal container (redundant)
  - Stop and remove uptime-kuma container (already in K8s)
  - Stop Docker Prometheus (migrate scrape configs first if needed)
  - Stop Docker Grafana (already in K8s with kube-prometheus-stack)
  - Document remaining Docker containers

  **Must NOT do**:
  - Do not remove container data directories yet (backup period)
  - Do not remove containers that haven't been migrated

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: SSH commands to stop containers
  - **Skills**: None needed

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 2 (with Tasks 3, 4)
  - **Blocks**: None
  - **Blocked By**: None

  **References**:

  **Current Docker instances to decommission**:
  - `nginx` - Broken (restart loop)
  - `nginx-internal` - Redundant with Traefik
  - `uptime` - Already running in K8s as uptime-kuma
  - `prometheus` - Replaced by kube-prometheus-stack
  - `grafana` - Replaced by kube-prometheus-stack

  **Acceptance Criteria**:

  ```bash
  # SSH to c3po and stop containers
  ssh c3po.home "docker stop nginx nginx-internal uptime prometheus grafana"
  # Expected: Containers stopped

  ssh c3po.home "docker rm nginx nginx-internal uptime prometheus grafana"
  # Expected: Containers removed

  # Verify K8s services still working
  curl -s https://grafana.k3s.s33g.uk/api/health
  # Expected: {"database":"ok"}

  curl -s https://uptime.k3s.s33g.uk
  # Expected: HTTP 200
  ```

  **Commit**: NO (operational task, no files changed)

---

### Phase 3: Media Stack Migration

- [ ] 6. Create Media Storage PVCs

  **What to do**:
  - Create PersistentVolume for //jocasta.home/Media using SMB CSI
  - Create PVC for media apps to share
  - Create separate PVCs for app configs (Longhorn)
  - Test mount with a debug pod

  **Must NOT do**:
  - Do not use ReadWriteOnce for shared media (need ReadWriteMany)
  - Do not create one giant PVC for everything (separate config from media)

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: YAML manifests, straightforward
  - **Skills**: [`git-master`]

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Sequential (after Task 1)
  - **Blocks**: Tasks 7-12 (all media apps)
  - **Blocked By**: Task 1 (SMB CSI)

  **References**:

  **Pattern References**:
  - SMB CSI PV example from Context7 research

  **External References**:
  - https://github.com/kubernetes-csi/csi-driver-smb/blob/master/deploy/example/e2e_usage.md

  **Storage Layout**:
  ```
  smb-media-pv -> //jocasta.home/Media (ReadWriteMany, 1Ti)
    ├── /Movies
    ├── /TV
    ├── /tmp
    └── /data

  Per-app configs (Longhorn, ReadWriteOnce):
    ├── plex-config-pvc (50Gi)
    ├── sonarr-config-pvc (5Gi)
    ├── radarr-config-pvc (5Gi)
    ├── prowlarr-config-pvc (2Gi)
    ├── sabnzbd-config-pvc (5Gi)
    └── etc.
  ```

  **Acceptance Criteria**:

  ```bash
  # Verify PV and PVC created
  kubectl get pv smb-media-pv
  # Expected: Bound

  kubectl get pvc -n media media-pvc
  # Expected: Bound to smb-media-pv

  # Test mount with debug pod
  kubectl run test-smb --image=busybox --rm -it --restart=Never \
    --overrides='{"spec":{"containers":[{"name":"test","image":"busybox","command":["ls","-la","/media"],"volumeMounts":[{"name":"media","mountPath":"/media"}]}],"volumes":[{"name":"media","persistentVolumeClaim":{"claimName":"media-pvc"}}]}}' \
    -n media
  # Expected: Lists Movies, TV, tmp, data directories
  ```

  **Files to create**:
  - `apps/media/namespace.yaml`
  - `apps/media/pv-smb-media.yaml`
  - `apps/media/pvc-media.yaml`

  **Commit**: YES
  - Message: `feat(media): create SMB storage for media stack`
  - Files: `apps/media/pv-*.yaml`, `apps/media/pvc-*.yaml`

---

- [ ] 7. Deploy Prowlarr (Indexer Manager)

  **What to do**:
  - Create ArgoCD Application using linuxserver/prowlarr image
  - Mount config PVC (Longhorn)
  - Mount shared media PVC for /data
  - Create IngressRoute for prowlarr.k3s.s33g.uk
  - Configure health checks

  **Must NOT do**:
  - Do not enable authentication initially (configure after migration)

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: [`git-master`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3 (with Tasks 8-12)
  - **Blocks**: None (other apps work independently)
  - **Blocked By**: Task 6 (Media PVCs)

  **References**:

  **Pattern References**:
  - `apps/uptime-kuma/application.yaml` - App with persistence
  - `infrastructure/traefik/config/ingress-uptime-kuma.yaml` - IngressRoute pattern
  - Ansible config: `/home/seeg/infra/ansible/roles/media/defaults/main/main.yml:129-150`

  **Current Docker config**:
  - Image: `lscr.io/linuxserver/prowlarr:1.35.1`
  - Port: 9696
  - Volumes: `/opt/prowlarr:/config`, `/mnt/media/data:/data`
  - Env: PUID=1000, PGID=1000, TZ=Europe/London

  **Acceptance Criteria**:

  ```bash
  # Verify pod running
  kubectl get pods -n media -l app=prowlarr
  # Expected: 1 pod Running

  # Verify ingress
  curl -sk https://prowlarr.k3s.s33g.uk/ping
  # Expected: "OK" or HTTP 200

  # Verify storage mounts
  kubectl exec -n media deploy/prowlarr -- ls /config
  # Expected: prowlarr.db and config files

  kubectl exec -n media deploy/prowlarr -- ls /data
  # Expected: Media data directory contents
  ```

  **Files to create**:
  - `apps/media/prowlarr/application.yaml`
  - `infrastructure/traefik/config/ingress-prowlarr.yaml`

  **Commit**: YES
  - Message: `feat(media): deploy prowlarr indexer manager`
  - Files: `apps/media/prowlarr/*`, `infrastructure/traefik/config/ingress-prowlarr.yaml`

---

- [ ] 8. Deploy SABnzbd (Usenet Downloader)

  **What to do**:
  - Create ArgoCD Application using home-operations/sabnzbd image
  - Mount config PVC and media PVC
  - Create IngressRoute for sabnzbd.k3s.s33g.uk
  - Store API key in 1Password via ExternalSecret

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: [`git-master`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3
  - **Blocked By**: Task 6

  **References**:

  **Current Docker config** (from Ansible):
  - Image: `ghcr.io/home-operations/sabnzbd:4.5.1`
  - Port: 8080
  - Volumes: `/opt/sabnzbd:/config`, `/mnt/media/data:/data`
  - Env: TZ=Europe/London, SABNZBD__API_KEY, SABNZBD__NZB_KEY

  **Acceptance Criteria**:

  ```bash
  kubectl get pods -n media -l app=sabnzbd
  # Expected: 1 pod Running

  curl -sk https://sabnzbd.k3s.s33g.uk
  # Expected: SABnzbd web UI
  ```

  **Files to create**:
  - `apps/media/sabnzbd/application.yaml`
  - `apps/media/sabnzbd/external-secret.yaml`
  - `infrastructure/traefik/config/ingress-sabnzbd.yaml`

  **Commit**: YES
  - Message: `feat(media): deploy sabnzbd usenet downloader`

---

- [ ] 9. Deploy Sonarr (TV Manager)

  **What to do**:
  - Create ArgoCD Application using linuxserver/sonarr
  - Mount config PVC, media PVC with /data and /tv paths
  - Create IngressRoute for sonarr.k3s.s33g.uk

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: [`git-master`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3
  - **Blocked By**: Task 6

  **References**:

  **Current Docker config**:
  - Image: `lscr.io/linuxserver/sonarr:4.0.14`
  - Port: 8989
  - Volumes: `/opt/sonarr:/config`, `/mnt/media/data:/data`, `/mnt/media/TV:/tv`

  **Acceptance Criteria**:

  ```bash
  kubectl get pods -n media -l app=sonarr
  # Expected: 1 pod Running, healthy

  curl -sk https://sonarr.k3s.s33g.uk/ping
  # Expected: "OK"
  ```

  **Files to create**:
  - `apps/media/sonarr/application.yaml`
  - `infrastructure/traefik/config/ingress-sonarr.yaml`

  **Commit**: YES
  - Message: `feat(media): deploy sonarr tv manager`

---

- [ ] 10. Deploy Radarr (Movie Manager)

  **What to do**:
  - Create ArgoCD Application using linuxserver/radarr
  - Mount config PVC, media PVC with /data and /movies paths
  - Create IngressRoute for radarr.k3s.s33g.uk

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: [`git-master`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3
  - **Blocked By**: Task 6

  **References**:

  **Current Docker config**:
  - Image: `lscr.io/linuxserver/radarr:5.23.3`
  - Port: 7878
  - Volumes: `/opt/radarr:/config`, `/mnt/media/data:/data`, `/mnt/media/Movies:/movies`

  **Acceptance Criteria**:

  ```bash
  kubectl get pods -n media -l app=radarr
  # Expected: 1 pod Running, healthy

  curl -sk https://radarr.k3s.s33g.uk/ping
  # Expected: "OK"
  ```

  **Files to create**:
  - `apps/media/radarr/application.yaml`
  - `infrastructure/traefik/config/ingress-radarr.yaml`

  **Commit**: YES
  - Message: `feat(media): deploy radarr movie manager`

---

- [ ] 11. Deploy Plex Media Server

  **What to do**:
  - Create ArgoCD Application using linuxserver/plex
  - Mount config PVC (larger - 50Gi for metadata)
  - Mount media PVC with /movies and /tv paths
  - Create IngressRoute for plex.k3s.s33g.uk
  - Store PLEX_CLAIM in 1Password for initial setup

  **Must NOT do**:
  - Do not enable hardware transcoding initially (add later if needed)

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: [`git-master`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3
  - **Blocked By**: Task 6

  **References**:

  **Current Docker config**:
  - Image: `lscr.io/linuxserver/plex:1.41.7`
  - Port: 32400
  - Volumes: `/opt/plex:/config`, `/mnt/media/Movies:/movies`, `/mnt/media/TV:/tv`
  - Env: PLEX_CLAIM (for initial setup)

  **Acceptance Criteria**:

  ```bash
  kubectl get pods -n media -l app=plex
  # Expected: 1 pod Running

  curl -sk https://plex.k3s.s33g.uk/web
  # Expected: Plex web UI redirect
  ```

  **Files to create**:
  - `apps/media/plex/application.yaml`
  - `apps/media/plex/external-secret.yaml`
  - `infrastructure/traefik/config/ingress-plex.yaml`

  **Commit**: YES
  - Message: `feat(media): deploy plex media server`

---

- [ ] 12. Deploy Supporting Media Apps (Unpackerr, Deemix, Metube, Pinchflat)

  **What to do**:
  - Create ArgoCD Applications for each:
    - Unpackerr (archive extraction)
    - Deemix (music downloader)
    - Metube (YouTube downloader)
    - Pinchflat (YouTube archiver)
  - Mount appropriate PVCs
  - Create IngressRoutes for web UIs

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: [`git-master`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 3
  - **Blocked By**: Task 6

  **References**:

  **Current Docker configs**:
  - Unpackerr: `ghcr.io/unpackerr/unpackerr:0.14.5` - no web UI
  - Deemix: `ghcr.io/bambanah/deemix:v4.3.3` - port 6595
  - Metube: `ghcr.io/alexta69/metube` - port 8081
  - Pinchflat: `ghcr.io/kieraneglin/pinchflat` - port 8945

  **Acceptance Criteria**:

  ```bash
  # Verify all pods running
  kubectl get pods -n media
  # Expected: unpackerr, deemix, metube, pinchflat all Running

  # Test web UIs
  curl -sk https://deemix.k3s.s33g.uk
  curl -sk https://metube.k3s.s33g.uk
  curl -sk https://pinchflat.k3s.s33g.uk
  # Expected: All return HTTP 200
  ```

  **Files to create**:
  - `apps/media/unpackerr/application.yaml`
  - `apps/media/deemix/application.yaml`
  - `apps/media/metube/application.yaml`
  - `apps/media/pinchflat/application.yaml`
  - IngressRoutes for deemix, metube, pinchflat

  **Commit**: YES
  - Message: `feat(media): deploy supporting media apps (unpackerr, deemix, metube, pinchflat)`

---

### Phase 4: Lower Priority

- [ ] 13. Deploy Go API + Redis

  **What to do**:
  - Create ArgoCD Application for go-api
  - Create Redis deployment (or use Redis Helm chart)
  - Configure service discovery between API and Redis
  - Create IngressRoute for api.k3s.s33g.uk
  - Mount logs volume if needed

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: [`git-master`]

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 4 (with Task 14)
  - **Blocked By**: None

  **References**:

  **Current Docker config**:
  - go-api: Custom `public-api` image, port 8080
  - go-redis: `redis:alpine`, port 6379

  **Acceptance Criteria**:

  ```bash
  kubectl get pods -n go-api
  # Expected: go-api and redis pods Running

  curl -sk https://api.k3s.s33g.uk/health
  # Expected: HTTP 200
  ```

  **Files to create**:
  - `apps/go-api/application.yaml`
  - `apps/go-api/redis.yaml`
  - `infrastructure/traefik/config/ingress-go-api.yaml`

  **Commit**: YES
  - Message: `feat(go-api): deploy go api with redis to kubernetes`

---

- [ ] 14. Final Cleanup - Stop Migrated Docker Containers

  **What to do**:
  - After verifying all media apps work in K8s:
    - Stop Docker containers: plex, sonarr, radarr, prowlarr, sabnzbd, etc.
    - Stop cloudflared, syslog-ng, speedtest-exporter
    - Stop go-api, go-redis
  - Document final state
  - Update Ansible playbooks to skip migrated containers

  **Must NOT do**:
  - Do not delete /opt/*/config directories (keep backups)

  **Recommended Agent Profile**:
  - **Category**: `quick`
  - **Skills**: None

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 4 (with Task 13)
  - **Blocked By**: Tasks 7-12 verified working

  **Acceptance Criteria**:

  ```bash
  # Stop media containers
  ssh c3po.home "docker stop plex sonarr radarr prowlarr sabnzbd unpackerr deemix metube pinchflat"
  ssh c3po.home "docker stop cloudflared syslog-ng speedtest-exporter"
  ssh c3po.home "docker stop go-api go-redis"

  # Verify remaining Docker containers (should only be: fivem, netbootxyz, nginx_preseed, node_exporter, smartctl-exporter, cloudflare-ddns)
  ssh c3po.home "docker ps --format 'table {{.Names}}'"
  # Expected: Only non-migrated containers remain

  # Verify K8s services still operational
  kubectl get applications -n argocd
  # Expected: All Synced and Healthy
  ```

  **Commit**: NO (operational task)

---

## Commit Strategy

| After Task | Message | Files | Verification |
|------------|---------|-------|--------------|
| 1 | `feat(smb-csi): install SMB CSI driver for NAS storage access` | infrastructure/smb-csi/* | kubectl get sc smb-media |
| 2 | `feat(cloudflared): migrate cloudflare tunnel to kubernetes` | apps/cloudflared/* | kubectl logs -l app=cloudflared |
| 3 | `feat(monitoring): add speedtest-exporter to kubernetes` | apps/speedtest-exporter/* | curl localhost:9798/metrics |
| 4 | `feat(syslog-ng): deploy centralized syslog to kubernetes` | apps/syslog-ng/* | kubectl get svc -n syslog-ng |
| 6 | `feat(media): create SMB storage for media stack` | apps/media/*.yaml | kubectl get pvc -n media |
| 7-12 | `feat(media): deploy <app>` | apps/media/<app>/* | curl https://<app>.k3s.s33g.uk |
| 13 | `feat(go-api): deploy go api with redis to kubernetes` | apps/go-api/* | curl https://api.k3s.s33g.uk |

---

## Success Criteria

### Verification Commands

```bash
# Phase 1 verification
kubectl get pods -n kube-system -l app.kubernetes.io/name=csi-driver-smb  # CSI running
kubectl get pods -n cloudflared  # Cloudflared running

# Phase 2 verification
kubectl get pods -n monitoring -l app=speedtest-exporter  # Exporter running
kubectl get pods -n syslog-ng  # Syslog running

# Phase 3 verification
kubectl get pods -n media  # All media apps running
for app in prowlarr sabnzbd sonarr radarr plex deemix metube pinchflat; do
  curl -sk "https://${app}.k3s.s33g.uk" -o /dev/null -w "%{http_code} ${app}\n"
done
# Expected: All return 200

# Final state verification
ssh c3po.home "docker ps --format '{{.Names}}'" | wc -l  # Minimal containers remain
```

### Final Checklist

- [ ] SMB CSI driver operational with NAS access
- [ ] All media apps accessible via *.k3s.s33g.uk
- [ ] Plex can stream media from NAS
- [ ] Sonarr/Radarr can write to NAS
- [ ] Cloudflared tunnel serving K8s services
- [ ] Old Docker containers stopped (except FiveM, netbootxyz, exporters)
- [ ] ArgoCD shows all applications healthy
- [ ] No secrets hardcoded in Git

---

## Services NOT Migrating (Final Reference)

| Service | Reason |
|---------|--------|
| FiveM | Game server with host networking, UDP, complex ports |
| NetbootXYZ | PXE/TFTP requires UDP 69, host-level networking |
| nginx_preseed | Paired with NetbootXYZ |
| node_exporter | Host-specific metrics, runs per-host via Ansible |
| smartctl-exporter | Requires privileged disk access, host-specific |
| cloudflare-ddns | Updates different DNS records than external-dns |
