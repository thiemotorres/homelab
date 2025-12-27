# Tandoor Recipes Integration Design

**Date:** 2025-12-27
**Status:** Approved
**Author:** Claude Code

## Overview

Integration of Tandoor Recipes (recipe management application) into the homelab Kubernetes cluster, accessible at `rezepte.thiemo.click` with German language settings and PostgreSQL backend.

## Requirements

- **URL:** rezepte.thiemo.click
- **Language:** German (TZ=Europe/Berlin)
- **Database:** Existing PostgreSQL instance in infrastructure namespace
- **Storage:** Hetzner Storage Box via SMB-CSI for media files
- **Access:** Public with Tandoor's built-in authentication
- **Configuration:** Standard setup with all features enabled
- **Resources:** Small footprint (512Mi RAM, 250m CPU limits)

## Architecture

### Namespace & RBAC

- **Namespace:** `tandoor`
- **ServiceAccount:** `tandoor-sa` with minimal privileges
- **RBAC:** Role and RoleBinding for pod management (follows n8n/gork pattern)

### Database

- **Type:** PostgreSQL (shared infrastructure instance)
- **Database Name:** `tandoor`
- **Host:** `postgres-service.infrastructure.svc.cluster.local:5432`
- **Credentials:** Stored in Sealed Secrets
- **Schema:** Automatically created by Tandoor Django migrations on first run

### Storage

**Persistent Volume Claim:**
- **Name:** `hetzner-tandoor-pvc`
- **StorageClass:** `hetzner-smb` (SMB-CSI Driver)
- **Size:** 10Gi
- **Access Mode:** ReadWriteOnce
- **Purpose:** Recipe images, media uploads, static files

**Mount Points:**
- `/opt/recipes/mediafiles` - Recipe images and user uploads
- `/opt/recipes/staticfiles` - Django static files

### Deployment Configuration

**Container Image:**
- `vabene1111/recipes:latest`
- Port: 8080

**Init Containers:**
1. **volume-permissions** - Sets correct ownership (node user)
2. **wait-for-postgres** - Ensures database is ready before startup

**Environment Variables:**

| Variable | Value | Source |
|----------|-------|--------|
| SECRET_KEY | Random 50+ chars | Sealed Secret |
| DB_ENGINE | django.db.backends.postgresql | ConfigMap/Direct |
| POSTGRES_HOST | postgres-service.infrastructure.svc.cluster.local | Direct |
| POSTGRES_PORT | 5432 | Direct |
| POSTGRES_DB | tandoor | Direct |
| POSTGRES_USER | (postgres user) | Sealed Secret |
| POSTGRES_PASSWORD | (postgres password) | Sealed Secret |
| ALLOWED_HOSTS | rezepte.thiemo.click | Direct |
| CSRF_TRUSTED_ORIGINS | https://rezepte.thiemo.click | Direct |
| TZ | Europe/Berlin | Direct |
| ENABLE_SIGNUP | 0 | Direct |
| ENABLE_PDF_EXPORT | 1 | Direct |

**Resource Limits:**
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

**Health Checks:**
- **Liveness Probe:** HTTP GET / on port 8080 (initial delay: 30s)
- **Readiness Probe:** HTTP GET / on port 8080 (initial delay: 10s)

### Networking

**Service:**
- **Name:** `tandoor-service`
- **Type:** ClusterIP
- **Port:** 8080 → 8080

**Certificate (cert-manager):**
- **Name:** `tandoor-cert`
- **Domain:** rezepte.thiemo.click
- **Issuer:** letsencrypt-prod (ClusterIssuer)
- **Secret:** tandoor-tls

**IngressRoute (Traefik):**
- **EntryPoint:** websecure (HTTPS)
- **Host Match:** `Host(\`rezepte.thiemo.click\`)`
- **Service:** tandoor-service:8080
- **TLS:** Enabled via tandoor-tls secret
- **HTTP→HTTPS Redirect:** Automatic via Traefik

## Security

**Secrets Management:**
1. `postgres-credentials` - PostgreSQL username/password (copied from infrastructure)
2. `tandoor-secret-key` - Django SECRET_KEY (generated using `openssl rand -base64 50`)

All secrets encrypted using Sealed Secrets for GitOps workflow.

**Access Control:**
- No additional authentication layer (relies on Tandoor's built-in user management)
- ENABLE_SIGNUP=0 prevents public registration
- Admin creates user accounts manually in Tandoor UI

## File Structure

```
apps/tandoor/
├── namespace.yaml                      # Namespace definition
├── rbac.yaml                          # ServiceAccount, Role, RoleBinding
├── pvc.yaml                           # Hetzner Storage PVC
├── deployment.yaml                    # Tandoor deployment with init containers
├── service.yaml                       # ClusterIP service
├── certificate.yaml                   # cert-manager Certificate
├── ingressroute.yaml                  # Traefik IngressRoute
├── kustomization.yaml                 # Kustomize configuration
└── sealed-secrets/
    ├── postgres-credentials-sealed.yaml   # DB credentials
    └── tandoor-secret-key-sealed.yaml     # Django SECRET_KEY
```

## Deployment Steps

### 1. Create Sealed Secrets

```bash
# Generate Tandoor SECRET_KEY
SECRET_KEY=$(openssl rand -base64 50)

# Create sealed secret for SECRET_KEY
kubectl create secret generic tandoor-secret-key \
  --from-literal=secret-key="$SECRET_KEY" \
  --namespace tandoor \
  --dry-run=client -o yaml | \
kubeseal --controller-name=sealed-secrets --controller-namespace=kube-system \
  --format yaml > apps/tandoor/sealed-secrets/tandoor-secret-key-sealed.yaml

# Copy postgres credentials (use existing sealed secret from infrastructure)
# Create sealed secret in tandoor namespace
kubectl get secret postgres-credentials -n infrastructure -o yaml | \
  sed 's/namespace: infrastructure/namespace: tandoor/' | \
  kubeseal --controller-name=sealed-secrets --controller-namespace=kube-system \
  --format yaml > apps/tandoor/sealed-secrets/postgres-credentials-sealed.yaml
```

### 2. Create PostgreSQL Database

```bash
# Connect to PostgreSQL pod
kubectl exec -it -n infrastructure postgres-0 -- psql -U <postgres-user>

# Create database
CREATE DATABASE tandoor;
\q
```

### 3. Deploy via GitOps

```bash
# Add tandoor to apps/kustomization.yaml
# Commit and push
git add apps/tandoor/
git add apps/kustomization.yaml
git commit -m "feat: add Tandoor Recipes application"
git push

# ArgoCD syncs automatically within ~3 minutes
# Monitor deployment
kubectl get pods -n tandoor -w
```

### 4. Verify Deployment

```bash
# Check pod status
kubectl get pods -n tandoor

# Check certificate
kubectl get certificate -n tandoor

# Check ingress
kubectl get ingressroute -n tandoor

# View logs
kubectl logs -n tandoor deployment/tandoor

# Access application
# https://rezepte.thiemo.click
```

### 5. Initial Setup

1. Navigate to `https://rezepte.thiemo.click`
2. Create admin account on first login
3. Configure language to German in settings
4. Start adding recipes!

## Integration with Existing Infrastructure

**Updates Required:**
1. `apps/kustomization.yaml` - Add `- tandoor` to resources list
2. Optional: Create ArgoCD Application manifest (or rely on root-app auto-discovery)

**No Changes Required:**
- PostgreSQL (uses existing instance)
- Traefik (existing IngressRoute pattern)
- cert-manager (existing ClusterIssuer)
- Sealed Secrets (existing controller)

## Rollback Plan

```bash
# Remove from kustomization
# Edit apps/kustomization.yaml and remove '- tandoor'

# Delete ArgoCD application (if created)
kubectl delete application tandoor -n argocd

# Delete namespace (cascades all resources)
kubectl delete namespace tandoor

# Drop database (if desired)
kubectl exec -it -n infrastructure postgres-0 -- psql -U <user> -c "DROP DATABASE tandoor;"
```

## Maintenance

**Updates:**
- Image updates via deployment.yaml modification
- ArgoCD auto-syncs changes from Git
- Database migrations run automatically on container start

**Backups:**
- Database included in existing PostgreSQL backup cronjob (R2)
- Media files on Hetzner Storage Box (covered by Hetzner backups)

**Monitoring:**
- Can enable Prometheus metrics via `ENABLE_METRICS=1` if needed
- Standard Kubernetes pod monitoring via existing Prometheus stack

## References

- [Tandoor Docker Documentation](https://docs.tandoor.dev/install/docker/)
- [Tandoor Configuration](https://docs.tandoor.dev/system/configuration/)
- [Tandoor GitHub Issue #3605](https://github.com/TandoorRecipes/recipes/issues/3605)

## Success Criteria

- ✅ Tandoor accessible at https://rezepte.thiemo.click
- ✅ TLS certificate valid (Let's Encrypt)
- ✅ PostgreSQL connection working
- ✅ Media files persist on Hetzner Storage
- ✅ German timezone configured
- ✅ Admin can create user accounts
- ✅ Recipe import/export functional
- ✅ PDF export enabled
