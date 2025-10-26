# GitOps with ArgoCD

This document describes the GitOps workflow using ArgoCD for declarative continuous deployment.

## Overview

The homelab uses **ArgoCD** to implement GitOps, providing:
- Git as the single source of truth
- Automated deployment from Git repository
- Self-healing when cluster state drifts
- Automatic pruning of deleted resources
- Declarative cluster configuration
- Visual UI for deployment status

## GitOps Principles

### What is GitOps?

**GitOps** is a way of managing infrastructure and applications where:

1. **Declarative**: Everything is defined in Git (YAML manifests)
2. **Versioned**: Git history tracks all changes
3. **Immutable**: Git commits are immutable references
4. **Automated**: Changes trigger automatic deployments
5. **Auditable**: Every change has a commit message and author

### Git as Source of Truth

```
┌─────────────────────┐
│   Git Repository    │
│  (thiemotorres/     │
│      homelab)       │
└──────────┬──────────┘
           │
           │ Watches for changes
           │
           ▼
┌──────────────────────┐
│      ArgoCD          │
│  - Compares Git      │
│  - Detects drift     │
│  - Auto-syncs        │
└──────────┬───────────┘
           │
           │ Applies manifests
           │
           ▼
┌──────────────────────┐
│  Kubernetes Cluster  │
│  - Deployments       │
│  - Services          │
│  - ConfigMaps        │
└──────────────────────┘
```

---

## Architecture

### App of Apps Pattern

The homelab uses the **App of Apps** pattern - a root ArgoCD Application that manages other Applications.

```
root-app (argocd-apps/root-app.yaml)
├── sealed-secrets (Helm: bitnami-labs/sealed-secrets)
├── cert-manager (Helm: jetstack/cert-manager)
├── traefik (Helm: traefik/traefik)
├── infrastructure (Kustomize: infrastructure/)
│   ├── namespaces
│   ├── cert-manager (ClusterIssuers, Certificates)
│   └── postgres (Database, Backups)
├── monitoring (Helm: kube-prometheus-stack)
├── monitoring-infra (Kustomize: infrastructure/monitoring/)
│   ├── Grafana ingress
│   ├── alertmanager-discord
│   └── Sealed secrets
└── apps (Kustomize: apps/)
    ├── gork (Backend + Frontend)
    ├── n8n (Workflow automation)
    └── external-services (Router proxy)
```

**Benefits:**
- Single source of truth (root-app)
- Hierarchical organization
- Clear dependencies
- Easy to enable/disable entire stacks

---

## ArgoCD Installation

### Bootstrap Process

ArgoCD is installed using Kustomize with Helm:

**Location**: `bootstrap/argocd/`

```bash
# Install ArgoCD
kubectl apply -k bootstrap/argocd/

# Wait for ArgoCD to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Deploy root application
kubectl apply -f argocd-apps/root-app.yaml
```

### ArgoCD Configuration

**File**: `bootstrap/argocd/kustomization.yaml`

```yaml
helmCharts:
- name: argo-cd
  repo: https://argoproj.github.io/argo-helm
  version: 7.7.10
  releaseName: argocd
  namespace: argocd
  valuesFile: values.yaml
```

**Key configurations** (`bootstrap/argocd/values.yaml`):

```yaml
configs:
  cm:
    # Metrics enabled for Prometheus
    metrics.enabled: "true"

    # Custom resource exclusions
    resource.exclusions: |
      - apiGroups:
        - "*"
        kinds:
        - Endpoints
        clusters:
        - "*"

  params:
    server.insecure: "true"  # TLS handled by Traefik

# Metrics for monitoring
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

---

## Application Definitions

### Root Application

**File**: `argocd-apps/root-app.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/thiemotorres/homelab.git
    targetRevision: main
    path: argocd-apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Key features:**
- Watches `argocd-apps/` directory
- Auto-syncs when Git changes
- Auto-prunes deleted resources
- Self-heals when cluster drifts

### Application Template

All child applications follow this pattern:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # Cleanup on deletion
spec:
  project: default

  source:
    # For Helm charts
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 65.7.0
    helm:
      valuesObject:
        # ... helm values

    # OR for Kustomize/plain manifests
    repoURL: https://github.com/thiemotorres/homelab.git
    targetRevision: main
    path: apps/gork

  destination:
    server: https://kubernetes.default.svc
    namespace: <target-namespace>

  syncPolicy:
    automated:
      prune: true      # Delete resources removed from Git
      selfHeal: true   # Fix drift automatically
    syncOptions:
      - CreateNamespace=true    # Create namespace if missing
      - ServerSideApply=true    # Use server-side apply (optional)
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## Sync Policies

### Automated Sync

All applications use automated sync:

```yaml
syncPolicy:
  automated:
    prune: true      # Remove resources deleted from Git
    selfHeal: true   # Auto-correct drift
```

**Behavior:**
- Git change → ArgoCD detects → Auto-syncs within ~3 minutes
- Cluster drift → ArgoCD detects → Auto-corrects
- Deleted file in Git → Resource deleted from cluster

### Manual Sync

If automated sync is disabled:

```bash
# Via CLI
argocd app sync <app-name>

# Via UI
# Click "Sync" button in ArgoCD UI
```

### Sync Options

**CreateNamespace**:
```yaml
syncOptions:
  - CreateNamespace=true  # ArgoCD creates namespace if missing
```

**ServerSideApply**:
```yaml
syncOptions:
  - ServerSideApply=true  # Use server-side apply (better for large resources)
```

Used for monitoring stack to handle large CRDs.

### Retry Policy

```yaml
retry:
  limit: 5              # Try up to 5 times
  backoff:
    duration: 5s        # Start with 5s delay
    factor: 2           # Double each retry (5s, 10s, 20s, 40s, 80s)
    maxDuration: 3m     # Max 3 minutes between retries
```

**Handles:**
- Transient network errors
- Dependency timing issues
- Temporary API server overload

---

## Application Types

### Helm Applications

**Example**: Monitoring Stack

```yaml
source:
  repoURL: https://prometheus-community.github.io/helm-charts
  chart: kube-prometheus-stack
  targetRevision: 65.7.0
  helm:
    valuesObject:
      prometheus:
        enabled: true
      # ... full values inline
```

**Advantages:**
- Easy to use popular charts
- Version pinning
- Inline or external values files

**Helm apps in homelab:**
- sealed-secrets
- cert-manager
- traefik
- kube-prometheus-stack (monitoring)

### Kustomize Applications

**Example**: Gork Application

```yaml
source:
  repoURL: https://github.com/thiemotorres/homelab.git
  targetRevision: main
  path: apps/gork
```

**Requires** `kustomization.yaml` in the directory:

```yaml
# apps/gork/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: gork

resources:
  - namespace.yaml
  - rbac.yaml
  - backend-deployment.yaml
  - frontend-deployment.yaml
  - service.yaml
  - certificate.yaml
  - ingressroute.yaml
  - sealed-secrets/
```

**Advantages:**
- Template-free (plain YAML)
- GitOps-friendly
- Namespace scoping
- Resource aggregation

**Kustomize apps in homelab:**
- infrastructure (namespaces, postgres)
- monitoring-infra (Grafana ingress, alertmanager-discord)
- apps (gork, n8n, external-services)

---

## Accessing ArgoCD

### Web UI

**URL**: https://argocd.feto.dev

**Login:**
```bash
# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d

# Username: admin
# Password: <from above>
```

**Features:**
- Visual app status
- Sync/refresh buttons
- Resource tree view
- Event logs
- Diff view

### CLI

**Install argocd CLI:**
```bash
# macOS
brew install argocd

# Linux
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

**Login:**
```bash
argocd login argocd.feto.dev
# Username: admin
# Password: <admin password>
```

**Common commands:**
```bash
# List applications
argocd app list

# Get app details
argocd app get <app-name>

# Sync app
argocd app sync <app-name>

# View diff
argocd app diff <app-name>

# View logs
argocd app logs <app-name>

# Delete app
argocd app delete <app-name>
```

---

## Workflow: Making Changes

### 1. Make Changes to Git

```bash
# Edit files locally
vim apps/gork/backend-deployment.yaml

# Commit changes
git add apps/gork/backend-deployment.yaml
git commit -m "Update gork-backend image to v1.1.0"

# Push to main branch
git push origin main
```

### 2. ArgoCD Detects Changes

**Automatic detection** (~3 minutes):
- ArgoCD polls Git repository
- Detects commit hash changed
- Shows app as "OutOfSync"

**Manual refresh:**
```bash
argocd app get <app-name> --refresh
```

### 3. Automatic Sync

With `automated: true`:
- ArgoCD automatically applies changes
- Updates Kubernetes resources
- Shows as "Synced" when complete

### 4. Verify Deployment

**Via UI:**
- Check app status in ArgoCD UI
- View resource tree
- Check for errors

**Via CLI:**
```bash
# Check app status
argocd app get gork

# Check pods restarted
kubectl get pods -n gork

# View logs
kubectl logs -n gork deployment/gork-backend
```

---

## Sync Waves and Hooks

### Sync Waves

Control deployment order with annotations:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Deploy in wave 1
```

**Example use case:**
```yaml
# Wave 0: Namespaces
- namespace.yaml (wave: 0)

# Wave 1: Secrets
- sealed-secrets/ (wave: 1)

# Wave 2: Deployments
- deployment.yaml (wave: 2)
```

**Default**: All resources in wave 0 if not specified.

### Sync Hooks

Execute tasks at specific sync phases:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync  # Run before sync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded  # Delete after success
```

**Hook types:**
- `PreSync` - Before sync
- `Sync` - During sync
- `PostSync` - After sync
- `SyncFail` - On sync failure

**Example**: Database migration job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: migrate-tool
        command: ["migrate", "up"]
```

---

## Health Assessment

ArgoCD determines application health based on resource status.

### Healthy Resources

- **Deployment**: All replicas ready
- **Pod**: Running and ready
- **Service**: Endpoints exist
- **Job**: Completed successfully
- **PVC**: Bound

### Unhealthy Resources

- **Deployment**: Replicas not ready
- **Pod**: CrashLoopBackOff, Error
- **Job**: Failed
- **PVC**: Pending

### Progressing

- **Deployment**: Rolling update in progress
- **Job**: Running

### Custom Health Checks

For custom resources:

```yaml
# In values.yaml
configs:
  cm:
    resource.customizations.health.<group>_<kind>: |
      hs = {}
      if obj.status ~= nil then
        if obj.status.phase == "Ready" then
          hs.status = "Healthy"
        end
      end
      return hs
```

---

## Troubleshooting

### App OutOfSync

**Symptoms**: App shows "OutOfSync" in UI

**Causes:**
- Manual changes in cluster
- Git changes not yet synced
- Ignored resources differ

**Solutions:**
```bash
# Check diff
argocd app diff <app-name>

# Force sync
argocd app sync <app-name> --force

# Hard refresh
argocd app get <app-name> --hard-refresh
```

### Sync Failing

**Check sync status:**
```bash
argocd app get <app-name>
```

**Common issues:**

1. **Invalid YAML**
   ```
   Solution: Validate YAML syntax
   kubectl apply --dry-run=client -f file.yaml
   ```

2. **Missing dependencies**
   ```
   Solution: Check sync waves, ensure dependencies exist
   ```

3. **Resource conflicts**
   ```
   Solution: Check for duplicate resources
   kubectl get <resource> -A
   ```

4. **Permission denied**
   ```
   Solution: Check ArgoCD ServiceAccount permissions
   ```

### App Stuck Progressing

**Symptoms**: App shows "Progressing" for extended time

**Causes:**
- Pods not starting
- Image pull errors
- Resource constraints

**Debug:**
```bash
# Check pods
kubectl get pods -n <namespace>

# Check events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Describe problematic resource
kubectl describe pod <pod-name> -n <namespace>
```

### Self-Healing Not Working

**Check syncPolicy:**
```yaml
syncPolicy:
  automated:
    selfHeal: true  # Must be true
```

**Manual drift correction:**
```bash
# View diff
argocd app diff <app-name>

# Sync to correct drift
argocd app sync <app-name>
```

---

## Best Practices

### 1. Use App of Apps Pattern

✅ Hierarchical organization
✅ Single root application
✅ Easy to manage dependencies

### 2. Enable Automated Sync

✅ Continuous deployment
✅ Fast feedback loop
✅ Reduced manual intervention

**But be careful:**
- Test in staging first
- Use sync waves for order
- Monitor for issues

### 3. Enable Pruning

✅ Keeps cluster clean
✅ Removes deleted resources
✅ Prevents resource accumulation

**Warning:**
- Can delete resources if Git file accidentally deleted
- Use Git carefully

### 4. Use Finalizers

```yaml
metadata:
  finalizers:
    - resources-finalizer.argocd.argoproj.io
```

✅ Proper cleanup on app deletion
✅ Cascading delete of resources

### 5. Pin Versions

```yaml
# Helm charts
targetRevision: 65.7.0  # Not "latest"

# Git repos
targetRevision: main  # Or specific commit SHA for immutability
```

### 6. Use Kustomize for Structure

✅ Template-free
✅ Clear resource organization
✅ Easy to review changes

### 7. Meaningful Commit Messages

```bash
# Bad
git commit -m "fix"

# Good
git commit -m "fix: Update gork-backend image to v1.1.0 to resolve memory leak"
```

---

## Monitoring ArgoCD

### Metrics

ArgoCD exposes Prometheus metrics:

**Key metrics:**
- `argocd_app_sync_total` - Total syncs per app
- `argocd_app_info` - App info (health, sync status)
- `argocd_app_sync_duration_seconds` - Sync duration

**ServiceMonitor enabled:**
```yaml
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

### Alerts

**Recommended alerts:**

```yaml
# Example PrometheusRule
- alert: ArgoCDAppOutOfSync
  expr: argocd_app_info{sync_status!="Synced"} == 1
  for: 15m
  annotations:
    summary: "ArgoCD app {{ $labels.name }} out of sync"

- alert: ArgoCDAppUnhealthy
  expr: argocd_app_info{health_status!="Healthy"} == 1
  for: 10m
  annotations:
    summary: "ArgoCD app {{ $labels.name }} unhealthy"
```

### Logging

```bash
# ArgoCD server logs
kubectl logs -n argocd deployment/argocd-server

# Application controller logs
kubectl logs -n argocd deployment/argocd-application-controller

# Repo server logs
kubectl logs -n argocd deployment/argocd-repo-server
```

---

## Security Considerations

### 1. Git Repository Access

**Current**: Public GitHub repository
**Risk**: Anyone can view configuration

**Mitigations:**
- Sensitive data in Sealed Secrets (encrypted)
- No plain-text passwords in Git
- Repository URL visible but contents declarative

**For private repos:**
```bash
argocd repo add https://github.com/user/private-repo \
  --username <username> \
  --password <token>
```

### 2. ArgoCD Admin Access

✅ TLS via cert-manager
✅ Admin password in Kubernetes secret
❌ No SSO/OIDC (homelab acceptable)

**Change admin password:**
```bash
# Via CLI
argocd account update-password

# Or via bcrypt hash in secret
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {"admin.password": "'$(htpasswd -bnBC 10 "" <password> | tr -d ':\n')'"}}'
```

### 3. RBAC

ArgoCD has its own RBAC system:

**Default**: admin has full access

**Add read-only user:**
```yaml
# In values.yaml
configs:
  cm:
    accounts.readonly: apiKey
  rbac:
    policy.csv: |
      p, role:readonly, applications, get, */*, allow
      g, readonly, role:readonly
```

### 4. Webhook Security

**GitHub webhook** (if using push-based sync):
- Configure webhook secret
- Restrict IP ranges
- Use HTTPS only

---

## Disaster Recovery

### Backup ArgoCD Configuration

**Applications are defined in Git** - no backup needed!

**ArgoCD itself:**
```bash
# Backup ArgoCD resources
kubectl get applications -n argocd -o yaml > argocd-apps-backup.yaml
kubectl get appprojects -n argocd -o yaml > argocd-projects-backup.yaml
```

### Restore ArgoCD

```bash
# 1. Reinstall ArgoCD
kubectl apply -k bootstrap/argocd/

# 2. Apply root application
kubectl apply -f argocd-apps/root-app.yaml

# 3. Wait for sync
# Everything else is restored automatically from Git!
```

**This is the beauty of GitOps** - disaster recovery is just re-pointing to Git.

---

## Advanced Features

### Multi-Source Applications

Deploy from multiple Git repos/Helm repos:

```yaml
sources:
  - repoURL: https://github.com/user/repo1
    path: base
  - repoURL: https://github.com/user/repo2
    path: overlays/prod
```

### App Sets (ApplicationSets)

Generate multiple applications from templates:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-apps
spec:
  generators:
  - list:
      elements:
      - cluster: prod
      - cluster: staging
  template:
    metadata:
      name: '{{cluster}}-app'
    spec:
      source:
        path: 'clusters/{{cluster}}'
```

---

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [GitOps Principles](https://www.gitops.tech/)
- [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)

---

## Changelog

- **2025-10-26**: Documented existing ArgoCD GitOps setup
  - App of Apps pattern with root-app
  - Automated sync with self-healing
  - Helm and Kustomize application types
  - Retry policies and sync waves
