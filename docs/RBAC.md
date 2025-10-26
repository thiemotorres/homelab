# RBAC Implementation Summary

This document describes the Role-Based Access Control (RBAC) configuration implemented across the homelab cluster.

## Overview

RBAC has been implemented following the **principle of least privilege** - each application receives only the minimum permissions necessary to function.

## Implementation Details

### Gork Namespace (`apps/gork/rbac.yaml`)

#### gork-backend
- **ServiceAccount**: `gork-backend-sa`
- **Permissions**: Read-only access to specific secrets
  - `postgres-credentials` - Database username/password
  - `database-url` - Database connection string
  - `gork-jwt-secret` - JWT signing key
- **Rationale**: Backend needs database access and JWT secret for authentication

#### gork-frontend
- **ServiceAccount**: `gork-frontend-sa`
- **Permissions**: None (empty role)
- **Rationale**: Static frontend has no need to access Kubernetes API

---

### N8N Namespace (`apps/n8n/rbac.yaml`)

#### n8n
- **ServiceAccount**: `n8n-sa`
- **Permissions**: Read-only access to specific secrets
  - `postgres-credentials` - Database credentials
- **Rationale**: n8n only needs database access; workflow definitions stored in persistent volume
- **Note**: ConfigMap permissions commented out but available if needed for workflow storage

---

### Infrastructure Namespace (`infrastructure/postgres/rbac.yaml`)

#### PostgreSQL
- **ServiceAccount**: `postgres-sa`
- **Permissions**: Read-only access to specific secrets
  - `postgres-credentials` - Own credentials
- **Rationale**: Minimal permissions for database operation

#### Backup Jobs (CronJob + Manual Jobs)
- **ServiceAccount**: `postgres-backup-sa`
- **Permissions**: Read-only access to specific secrets
  - `postgres-credentials` - Database access for dumps
  - `r2-backup-credentials` - Cloudflare R2 storage credentials
  - `discord-webhook` - Notification webhook URL
- **Rationale**: Backup jobs need database, storage, and notification access

---

### Monitoring Namespace (`infrastructure/monitoring/rbac.yaml`)

#### alertmanager-discord
- **ServiceAccount**: `alertmanager-discord-sa`
- **Permissions**: Read-only access to specific secrets
  - `discord-alerts` - Discord webhook URL
- **Rationale**: Only needs webhook URL to forward alerts

#### Prometheus, Grafana, Alertmanager
- **ServiceAccounts**: Managed by kube-prometheus-stack Helm chart
- **Permissions**: Configured by Helm chart values
- **Note**: These use the chart's default ServiceAccounts with appropriate permissions

---

## Security Benefits

### Before RBAC Implementation
- All pods used the `default` ServiceAccount
- Default ServiceAccount has minimal but unnecessary permissions
- If a pod was compromised, attacker could:
  - List pods and services cluster-wide
  - Potentially read ConfigMaps
  - Attempt privilege escalation

### After RBAC Implementation
- Each pod has its own ServiceAccount with specific permissions
- Pods can only access the exact secrets they need
- Compromised container has extremely limited access
- Blast radius is minimized
- Audit trail is clearer (actions tied to specific ServiceAccounts)

---

## Verification Commands

### Check ServiceAccount permissions
```bash
# List all ServiceAccounts in a namespace
kubectl get serviceaccounts -n gork

# Check what a specific ServiceAccount can do
kubectl auth can-i --list --as=system:serviceaccount:gork:gork-backend-sa -n gork

# Test specific permission
kubectl auth can-i get secrets --as=system:serviceaccount:gork:gork-backend-sa -n gork
kubectl auth can-i get secrets --as=system:serviceaccount:gork:gork-frontend-sa -n gork
```

### View RBAC resources
```bash
# List Roles in a namespace
kubectl get roles -n gork

# Describe a Role to see permissions
kubectl describe role gork-backend-role -n gork

# List RoleBindings
kubectl get rolebindings -n gork

# Describe RoleBinding
kubectl describe rolebinding gork-backend-binding -n gork
```

### Verify pod is using correct ServiceAccount
```bash
kubectl get pod <pod-name> -n gork -o jsonpath='{.spec.serviceAccountName}'
```

---

## Deployment Notes

### Changes Made to Deployments
Each deployment has been updated to specify `serviceAccountName`:

**Gork:**
- `apps/gork/backend-deployment.yaml` → Uses `gork-backend-sa`
- `apps/gork/frontend-deployment.yaml` → Uses `gork-frontend-sa`

**N8N:**
- `apps/n8n/n8n-deployment.yaml` → Uses `n8n-sa`

**Infrastructure:**
- `infrastructure/postgres/postgres.yaml` → Uses `postgres-sa`
- `infrastructure/postgres/backup-cronjob.yaml` → Uses `postgres-backup-sa`
- `infrastructure/postgres/manual-backup-job.yaml` → Uses `postgres-backup-sa`
- `infrastructure/postgres/manual-restore-job.yaml` → Uses `postgres-backup-sa`

**Monitoring:**
- `infrastructure/monitoring/alertmanager-discord.yaml` → Uses `alertmanager-discord-sa`

### Kustomization Updates
Each namespace's `kustomization.yaml` has been updated to include `rbac.yaml`:
- `apps/gork/kustomization.yaml`
- `apps/n8n/kustomization.yaml`
- `infrastructure/postgres/kustomization.yaml`
- `infrastructure/monitoring/kustomization.yaml`

---

## Testing After Deployment

Once ArgoCD applies these changes, you can test that RBAC is working:

### 1. Verify pods are running with new ServiceAccounts
```bash
kubectl get pods -n gork -o custom-columns=NAME:.metadata.name,SA:.spec.serviceAccountName
kubectl get pods -n n8n -o custom-columns=NAME:.metadata.name,SA:.spec.serviceAccountName
kubectl get pods -n infrastructure -o custom-columns=NAME:.metadata.name,SA:.spec.serviceAccountName
kubectl get pods -n monitoring -o custom-columns=NAME:.metadata.name,SA:.spec.serviceAccountName
```

### 2. Test that gork-backend CAN access its secrets
```bash
# This should return "yes"
kubectl auth can-i get secret/postgres-credentials \
  --as=system:serviceaccount:gork:gork-backend-sa -n gork
```

### 3. Test that gork-frontend CANNOT access secrets
```bash
# This should return "no"
kubectl auth can-i get secrets \
  --as=system:serviceaccount:gork:gork-frontend-sa -n gork
```

### 4. Test that services still work
```bash
# Access your applications and verify they function normally
curl https://gork.thiemo.click
curl https://n8n.thiemo.click
curl https://grafana.feto.dev
```

---

## Troubleshooting

### Pods not starting after RBAC changes
If pods fail to start with permission errors:

1. **Check ServiceAccount exists**:
   ```bash
   kubectl get sa -n <namespace>
   ```

2. **Check Role and RoleBinding exist**:
   ```bash
   kubectl get role,rolebinding -n <namespace>
   ```

3. **Verify the deployment references correct ServiceAccount**:
   ```bash
   kubectl get deployment <name> -n <namespace> -o yaml | grep serviceAccountName
   ```

### Application errors related to secrets
If an application can't read its secrets:

1. **Verify the Role includes the secret**:
   ```bash
   kubectl describe role <role-name> -n <namespace>
   ```

2. **Check the secret name matches exactly**:
   ```bash
   kubectl get secrets -n <namespace>
   ```

3. **Verify RoleBinding is correct**:
   ```bash
   kubectl describe rolebinding <binding-name> -n <namespace>
   ```

---

## Future Enhancements

### Potential Additions
1. **Pod Security Standards**: Add PodSecurityPolicy or Pod Security Admission
2. **Network Policies**: Add NetworkPolicies for pod-to-pod isolation
3. **Audit Logging**: Enable Kubernetes audit logs to track API access
4. **Secret Rotation**: Implement automated secret rotation policies
5. **OPA/Gatekeeper**: Add policy enforcement with Open Policy Agent

### Monitoring RBAC
Consider adding Prometheus alerts for:
- Unauthorized API access attempts
- ServiceAccount permission denials
- Unexpected ServiceAccount usage

---

## References

- [Kubernetes RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [K3s Security Documentation](https://docs.k3s.io/security/hardening-guide)
- [RBAC Best Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)

---

## Changelog

- **2025-10-26**: Initial RBAC implementation across all namespaces
  - Gork: 2 ServiceAccounts (backend, frontend)
  - N8N: 1 ServiceAccount
  - Infrastructure: 2 ServiceAccounts (postgres, backup)
  - Monitoring: 1 ServiceAccount (alertmanager-discord)
