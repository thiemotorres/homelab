# Homelab

A production-grade Kubernetes-based homelab running on K3s with GitOps practices using ArgoCD.

## Overview

This repository contains the infrastructure and application configurations for a personal homelab cluster. The setup uses:

- **K3s** - Lightweight Kubernetes distribution
- **ArgoCD** - GitOps continuous delivery
- **Traefik** - Ingress controller and reverse proxy
- **cert-manager** - Automated TLS certificate management with Let's Encrypt
- **Prometheus Stack** - Monitoring, alerting, and dashboards
- **PostgreSQL** - Database with automated backups to Cloudflare R2
- **Sealed Secrets** - Encrypted secrets management for GitOps

## Quick Start

### 1. Install ArgoCD

```bash
kubectl apply -k bootstrap/argocd/
```

### 2. Deploy Everything

```bash
kubectl apply -f argocd-apps/root-app.yaml
```

ArgoCD will automatically deploy all infrastructure and applications from Git.

### 3. Access Services

- **ArgoCD**: https://argocd.feto.dev
- **Grafana**: https://grafana.feto.dev
- **Gork**: https://gork.thiemo.click
- **n8n**: https://n8n.thiemo.click

## Documentation

📚 **Comprehensive documentation is available in the [docs/](docs/) directory:**

- **[Documentation Home](docs/README.md)** - Start here for complete overview
- **[RBAC](docs/RBAC.md)** - Security and access control
- **[GitOps with ArgoCD](docs/GITOPS-WITH-ARGOCD.md)** - Deployment workflow
- **[Ingress and Routing](docs/INGRESS-AND-ROUTING.md)** - Traefik configuration
- **[Certificate Management](docs/CERTIFICATE-MANAGEMENT.md)** - Automated TLS
- **[Monitoring and Alerting](docs/MONITORING-AND-ALERTING.md)** - Prometheus & Grafana
- **[Database Backup & Restore](docs/DATABASE-BACKUP-RESTORE.md)** - PostgreSQL backups

## Architecture

```
Internet → Cloudflare DNS → Traefik Ingress → Applications → PostgreSQL
              ↓                                                   ↓
        Let's Encrypt TLS                              Automated Backups (R2)
```

## Technology Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| K3s | v1.33.5 | Kubernetes distribution |
| ArgoCD | 7.7.10 | GitOps deployment |
| Traefik | 37.2.0 | Ingress controller |
| cert-manager | v1.16.2 | TLS automation |
| kube-prometheus-stack | 65.7.0 | Monitoring |
| PostgreSQL | 17-alpine | Database |

## Repository Structure

```
homelab/
├── bootstrap/          # ArgoCD installation
├── argocd-apps/        # Application definitions (App of Apps pattern)
├── infrastructure/     # Core infrastructure (namespaces, postgres, monitoring)
├── apps/               # User applications (gork, n8n, external-services)
└── docs/               # Complete documentation
```

## Key Features

✅ **GitOps** - All configuration in Git, automated deployment
✅ **Automated TLS** - Let's Encrypt certificates via DNS-01 validation
✅ **Monitoring** - Prometheus, Grafana, Alertmanager with Discord notifications
✅ **Backups** - Daily PostgreSQL backups to Cloudflare R2 with 7-day retention
✅ **RBAC** - Least-privilege security for all applications
✅ **Multi-domain** - Separate domains for internal (feto.dev) and public (thiemo.click) services

## Common Commands

```bash
# View all applications
kubectl get applications -n argocd

# Check certificates
kubectl get certificate -A

# View monitoring
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090

# Manual database backup
kubectl apply -f infrastructure/postgres/manual-backup-job.yaml

# View logs
kubectl logs -n <namespace> deployment/<name>
```

## Making Changes

1. Edit files in Git
2. Commit and push
3. ArgoCD automatically syncs (within ~3 minutes)
4. Verify in ArgoCD UI or via kubectl

See [GitOps documentation](docs/GITOPS-WITH-ARGOCD.md) for detailed workflow.

## Support

For detailed information, troubleshooting, and guides, see the [documentation](docs/README.md).

## License

Open source - feel free to use, modify, and learn from this configuration.

---

**Repository**: https://github.com/thiemotorres/homelab
**Maintainer**: Thiemo Torres (info@thiemo-torres.de)
**Last Updated**: 2025-10-26
