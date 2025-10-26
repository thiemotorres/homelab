# Homelab Documentation

Comprehensive documentation for the K3s-based homelab infrastructure.

## Overview

This homelab runs on K3s (lightweight Kubernetes) and uses GitOps principles with ArgoCD for declarative infrastructure management. The setup includes automated TLS certificates, monitoring with Prometheus/Grafana, PostgreSQL database with automated backups, and multiple applications.

## Quick Links

### How-To Guides

- **[Exposing External Services](guides/expose-external-services.md)** - Proxy external devices through Traefik with TLS

### Core Infrastructure

- **[RBAC (Role-Based Access Control)](RBAC.md)** - Security permissions for pods and applications
- **[GitOps with ArgoCD](GITOPS-WITH-ARGOCD.md)** - Continuous deployment from Git
- **[Ingress and Routing](INGRESS-AND-ROUTING.md)** - Traefik reverse proxy configuration
- **[Certificate Management](CERTIFICATE-MANAGEMENT.md)** - Automated TLS with Let's Encrypt
- **[Monitoring and Alerting](MONITORING-AND-ALERTING.md)** - Prometheus, Grafana, and Discord notifications
- **[Database Backup & Restore](DATABASE-BACKUP-RESTORE.md)** - PostgreSQL backup system

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Internet                           │
└────────────────────┬────────────────────────────────┘
                     │ HTTPS (443)
                     ▼
          ┌──────────────────────┐
          │  Cloudflare DNS      │
          │  + Let's Encrypt     │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │   Traefik Ingress    │
          │   (DaemonSet)        │
          └──────────┬───────────┘
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
┌───────────┐ ┌───────────┐ ┌───────────┐
│   Gork    │ │    n8n    │ │  Grafana  │
│ (Public)  │ │ (Public)  │ │(Internal) │
└─────┬─────┘ └─────┬─────┘ └─────┬─────┘
      │             │             │
      └─────────────┼─────────────┘
                    │
                    ▼
          ┌──────────────────────┐
          │    PostgreSQL        │
          │  - Automated backups │
          │  - 7-day retention   │
          └──────────────────────┘
```

## Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Kubernetes** | K3s v1.33.5 | Lightweight container orchestration |
| **GitOps** | ArgoCD 7.7.10 | Continuous deployment |
| **Ingress** | Traefik 37.2.0 | Reverse proxy & load balancer |
| **Certificates** | cert-manager v1.16.2 | Automated TLS from Let's Encrypt |
| **Secrets** | Sealed Secrets 2.16.2 | Encrypted secrets in Git |
| **Monitoring** | kube-prometheus-stack 65.7.0 | Metrics & alerting |
| **Database** | PostgreSQL 17 Alpine | Relational database |
| **Backup** | rclone + R2 | Automated backups to object storage |

## Cluster Information

- **Nodes**: 3 (1 control-plane, 2 workers)
- **OS**: Ubuntu 24.04.3 LTS
- **Container Runtime**: containerd 2.1.4
- **CNI**: Flannel (built-in K3s)
- **Network Policy**: K3s built-in controller
- **Storage**: local-path provisioner

## Deployed Applications

### Public Applications (thiemo.click)

| Application | URL | Description |
|------------|-----|-------------|
| **Gork** | https://gork.thiemo.click | Custom full-stack application |
| **n8n** | https://n8n.thiemo.click | Workflow automation platform |

### Internal Applications (feto.dev)

| Application | URL | Description |
|------------|-----|-------------|
| **ArgoCD** | https://argocd.feto.dev | GitOps deployment UI |
| **Grafana** | https://grafana.feto.dev | Monitoring dashboards |
| **Router** | https://router.feto.dev | Home router admin proxy |

## Getting Started

### Prerequisites

- K3s cluster (installation not covered here)
- kubectl configured
- Git repository access
- Cloudflare account (for DNS & certificates)

### Initial Setup

1. **Install ArgoCD**:
   ```bash
   kubectl apply -k bootstrap/argocd/
   ```

2. **Deploy Root Application**:
   ```bash
   kubectl apply -f argocd-apps/root-app.yaml
   ```

3. **Wait for Sync**:
   ```bash
   kubectl get applications -n argocd
   ```

4. **Access ArgoCD UI**:
   ```bash
   # Get admin password
   kubectl get secret argocd-initial-admin-secret -n argocd \
     -o jsonpath='{.data.password}' | base64 -d

   # Open https://argocd.feto.dev
   # Username: admin
   # Password: <from above>
   ```

### Making Changes

1. **Edit files in Git**:
   ```bash
   vim apps/gork/backend-deployment.yaml
   git commit -m "Update gork-backend image"
   git push
   ```

2. **ArgoCD automatically deploys** (within ~3 minutes)

3. **Verify**:
   ```bash
   argocd app get gork
   kubectl get pods -n gork
   ```

## Common Tasks

### View Application Logs

```bash
kubectl logs -n <namespace> deployment/<deployment-name>
```

### Scale Application

```bash
kubectl scale deployment <name> -n <namespace> --replicas=3
```

### Manual Backup

```bash
kubectl apply -f infrastructure/postgres/manual-backup-job.yaml
```

### Restore Database

```bash
kubectl create -f infrastructure/postgres/manual-restore-job.yaml
```

### View Metrics

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090

# Open http://localhost:9090
```

### Check Certificate Status

```bash
kubectl get certificate -A
kubectl describe certificate <name> -n <namespace>
```

## Directory Structure

```
homelab/
├── bootstrap/               # Initial cluster setup
│   └── argocd/             # ArgoCD installation
├── argocd-apps/            # ArgoCD Application definitions
│   ├── root-app.yaml       # App of Apps root
│   ├── sealed-secrets.yaml
│   ├── cert-manager.yaml
│   ├── traefik.yaml
│   ├── infrastructure.yaml
│   ├── monitoring.yaml
│   ├── monitoring-infra.yaml
│   └── apps.yaml
├── infrastructure/         # Core infrastructure
│   ├── namespaces/        # Namespace definitions
│   ├── cert-manager/      # ClusterIssuers, certificates
│   ├── postgres/          # Database & backups
│   └── monitoring/        # Monitoring infrastructure
├── apps/                  # User applications
│   ├── gork/             # Full-stack application
│   ├── n8n/              # Workflow automation
│   └── external-services/ # Service proxies
└── docs/                 # Documentation (you are here!)
    ├── README.md
    ├── RBAC.md
    ├── GITOPS-WITH-ARGOCD.md
    ├── INGRESS-AND-ROUTING.md
    ├── CERTIFICATE-MANAGEMENT.md
    ├── MONITORING-AND-ALERTING.md
    └── DATABASE-BACKUP-RESTORE.md
```

## Security

### Current Security Measures

✅ **RBAC**: Every pod has its own ServiceAccount with minimal permissions
✅ **TLS**: All external services use Let's Encrypt certificates
✅ **Sealed Secrets**: Secrets encrypted at rest in Git
✅ **Network Policies**: (Supported but not yet implemented)
✅ **Automated Updates**: GitOps ensures consistent configuration
✅ **Backup Encryption**: Backups stored in Cloudflare R2 with encryption

### Security Best Practices

1. **Change default passwords** after initial setup
2. **Rotate Cloudflare API tokens** annually
3. **Review RBAC permissions** regularly
4. **Monitor security alerts** in Discord
5. **Test backup restore** quarterly
6. **Keep ArgoCD updated** via Helm chart version

## Monitoring

### Key Metrics

Access Grafana at https://grafana.feto.dev to view:

- **Node metrics**: CPU, memory, disk, network
- **Pod metrics**: Resource usage, restarts
- **Application metrics**: Request rates, latencies
- **Storage metrics**: PVC usage

### Alerts

Alerts are sent to Discord for:

- High CPU/memory usage
- Pod crash loops
- Failed deployments
- Certificate expiration
- Backup failures
- Prometheus/Grafana issues

## Backup Strategy

### What Gets Backed Up

✅ **PostgreSQL databases** - Daily at 2 AM
✅ **Retention**: 7 days
✅ **Storage**: Cloudflare R2 (object storage)
✅ **Notifications**: Discord on success/failure

### What Doesn't Get Backed Up

❌ **Prometheus metrics** - Regenerate, large size
❌ **Logs** - Transient, not persistent
❌ **Container images** - Pulled from registries
❌ **Kubernetes state** - Defined in Git (GitOps)

### Disaster Recovery

**Full cluster loss:**
1. Reinstall K3s
2. Apply `bootstrap/argocd/`
3. Apply `argocd-apps/root-app.yaml`
4. Restore PostgreSQL from R2
5. Wait for ArgoCD to sync everything else

**Recovery Time Objective (RTO)**: ~30 minutes
**Recovery Point Objective (RPO)**: 24 hours (daily backups)

## Troubleshooting

### Application Not Accessible

1. **Check IngressRoute**:
   ```bash
   kubectl get ingressroute -n <namespace>
   ```

2. **Check Certificate**:
   ```bash
   kubectl get certificate -n <namespace>
   ```

3. **Check Traefik logs**:
   ```bash
   kubectl logs -n kube-system -l app.kubernetes.io/name=traefik
   ```

4. **Check pod status**:
   ```bash
   kubectl get pods -n <namespace>
   ```

### ArgoCD Not Syncing

1. **Check application status**:
   ```bash
   argocd app get <app-name>
   ```

2. **View sync logs**:
   ```bash
   argocd app logs <app-name>
   ```

3. **Force sync**:
   ```bash
   argocd app sync <app-name> --force
   ```

### Certificate Not Ready

1. **Check certificate**:
   ```bash
   kubectl describe certificate <cert-name> -n <namespace>
   ```

2. **Check cert-manager logs**:
   ```bash
   kubectl logs -n cert-manager deployment/cert-manager
   ```

3. **Check Cloudflare API token**:
   ```bash
   kubectl get secret cloudflare-api-token-* -n cert-manager
   ```

## Cost Analysis

### Monthly Costs

| Service | Cost | Notes |
|---------|------|-------|
| **Hardware** | $0 | Self-hosted |
| **Power** | ~$5-10 | 3x mini PCs @ ~50W |
| **Cloudflare R2** | ~$0.03 | 7 backups @ 50MB |
| **Domain (feto.dev)** | ~$1/month | Annual cost / 12 |
| **Domain (thiemo.click)** | ~$1/month | Annual cost / 12 |
| **Let's Encrypt** | $0 | Free TLS certificates |
| **Total** | ~$7-12/month | Extremely cost-effective |

## Performance

### Resource Usage (Typical)

| Component | Memory | CPU | Storage |
|-----------|--------|-----|---------|
| **K3s System** | ~500MB | ~0.5 | - |
| **Traefik** | ~100MB | ~0.1 | - |
| **Prometheus** | ~1.5GB | ~0.5 | 8GB |
| **Grafana** | ~200MB | ~0.1 | 2GB |
| **PostgreSQL** | ~300MB | ~0.2 | 5GB |
| **Applications** | ~500MB | ~0.3 | 10GB |
| **Total** | ~3GB | ~1.7 | ~25GB |

**Cluster Capacity**: 3 nodes with typical specs can easily handle this.

## Future Enhancements

### Planned

- [ ] **NetworkPolicies** - Network-level pod isolation
- [ ] **ServiceMonitors** - Custom application metrics
- [ ] **Custom Grafana Dashboards** - Application-specific dashboards
- [ ] **Pod Security Standards** - Enhanced pod security
- [ ] **Backup Verification** - Automated restore testing

### Under Consideration

- [ ] **GitOps for Secrets** - External Secrets Operator
- [ ] **Long-term Metrics** - Thanos for Prometheus
- [ ] **Log Aggregation** - Loki for centralized logs
- [ ] **Service Mesh** - Linkerd or Istio
- [ ] **Multi-cluster** - Expand to multiple environments

## Contributing

This is a personal homelab, but contributions are welcome!

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test in a dev environment
5. Submit a pull request

## References

### Official Documentation

- [K3s Documentation](https://docs.k3s.io/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

### Community Resources

- [r/kubernetes](https://reddit.com/r/kubernetes)
- [r/homelab](https://reddit.com/r/homelab)
- [K3s GitHub](https://github.com/k3s-io/k3s)
- [Awesome Kubernetes](https://github.com/ramitsurana/awesome-kubernetes)

## Support

For issues or questions:

1. Check the relevant documentation page
2. Review troubleshooting sections
3. Check ArgoCD UI for deployment status
4. Review pod logs and events
5. Open an issue on GitHub

## License

This homelab configuration is open source. Feel free to use, modify, and learn from it.

---

**Last Updated**: 2025-10-26
**Maintainer**: Thiemo Torres
**Repository**: https://github.com/thiemotorres/homelab
