# GitHub Actions CI/CD Pipelines

This directory contains GitHub Actions workflows for validating, securing, and maintaining the homelab infrastructure.

## Workflows Overview

### 1. Validation Pipeline ([validate.yml](validate.yml))

**Triggers**: Pull requests, pushes to main, manual dispatch

**Purpose**: Ensures all Kubernetes manifests and configurations are valid before deployment

**Jobs**:
- **validate-yaml**: Lints all YAML files for syntax errors
- **validate-kubernetes**: Validates Kubernetes manifests with `kubectl --dry-run`
- **validate-kustomize**: Builds and validates all Kustomize overlays
- **validate-helm**: Validates Helm charts and templates
- **validate-argocd-apps**: Checks ArgoCD Application definitions
- **check-no-secrets**: Ensures no plain secrets are committed (only SealedSecrets)
- **summary**: Provides a consolidated validation report

**Key Features**:
- Catches configuration errors before they reach the cluster
- Validates Kustomize builds without requiring cluster access
- Uploads built manifests as artifacts for review
- Fails if plain Kubernetes Secrets are found outside sealed-secrets directories

---

### 2. Security Scanning Pipeline ([security.yml](security.yml))

**Triggers**: Pull requests, pushes to main, weekly on Mondays at 9 AM UTC, manual dispatch

**Purpose**: Identifies security vulnerabilities and policy violations

**Jobs**:
- **secret-scanning**: Scans for leaked secrets using Gitleaks
- **kube-linter**: Checks Kubernetes security best practices
- **rbac-validation**: Analyzes RBAC configurations for overly permissive rules
- **kubesec-scan**: Security risk analysis for Kubernetes resources
- **network-policy-check**: Validates NetworkPolicy presence
- **resource-limits-check**: Ensures resource requests/limits are defined
- **container-security**: Checks securityContext configurations
- **summary**: Consolidated security report

**Key Features**:
- Prevents accidental secret commits
- Identifies overly permissive RBAC rules (wildcards, dangerous permissions)
- Recommends NetworkPolicies for defense-in-depth
- Validates that containers run as non-root with security contexts
- Checks for resource limits to prevent resource exhaustion

**Security Checks**:
- ✅ No plain secrets in Git
- ✅ RBAC follows least privilege principle
- ✅ Containers have resource limits
- ✅ Security contexts configured
- ⚠️ NetworkPolicies recommended

---

### 3. Documentation Pipeline ([docs.yml](docs.yml))

**Triggers**: Pull requests and pushes affecting docs/ or *.md files, manual dispatch

**Purpose**: Maintains documentation quality and consistency

**Jobs**:
- **markdown-lint**: Lints Markdown files for formatting consistency
- **link-checker**: Validates all links in documentation
- **check-documentation-structure**: Ensures essential docs exist
- **check-code-examples**: Validates YAML examples in documentation
- **check-consistency**: Checks version consistency between code and docs
- **spelling-check**: Spell-checks documentation (with technical dictionary)
- **summary**: Documentation validation report

**Key Features**:
- Catches broken links before users encounter them
- Validates code examples in documentation
- Checks that essential documentation files exist
- Identifies TODO/FIXME items for tracking
- Spell-checks with custom technical terms dictionary

**Essential Documentation**:
- ✅ [README.md](../../README.md)
- ✅ [docs/README.md](../../docs/README.md)
- ✅ [docs/RBAC.md](../../docs/RBAC.md)
- ✅ [docs/GITOPS-WITH-ARGOCD.md](../../docs/GITOPS-WITH-ARGOCD.md)
- ✅ [docs/INGRESS-AND-ROUTING.md](../../docs/INGRESS-AND-ROUTING.md)
- ✅ [docs/CERTIFICATE-MANAGEMENT.md](../../docs/CERTIFICATE-MANAGEMENT.md)
- ✅ [docs/MONITORING-AND-ALERTING.md](../../docs/MONITORING-AND-ALERTING.md)
- ✅ [docs/DATABASE-BACKUP-RESTORE.md](../../docs/DATABASE-BACKUP-RESTORE.md)

---

### 4. Scheduled Health Checks ([scheduled-checks.yml](scheduled-checks.yml))

**Triggers**: Weekly on Sundays at 3 AM UTC, manual dispatch

**Purpose**: Periodic validation of configurations and dependency health

**Jobs**:
- **backup-validation**: Validates backup CronJob configuration
- **certificate-check**: Reviews certificate configurations
- **monitoring-config-check**: Validates Prometheus/Grafana/Alertmanager setup
- **argocd-health-check**: Validates ArgoCD App of Apps structure
- **dependency-check**: Checks for outdated Helm chart versions
- **resource-review**: Reviews resource allocation across all deployments
- **summary**: Weekly health check report

**Key Features**:
- Validates backup schedule and retention policies
- Ensures disaster recovery procedures are in place
- Checks certificate configurations (production vs staging)
- Identifies outdated Helm chart dependencies
- Reviews resource allocations across all workloads
- Generates weekly health reports

**What It Checks**:
- ✅ Backup CronJob schedule and configuration
- ✅ Manual restore job availability
- ✅ Certificate ClusterIssuers (production & staging)
- ✅ Monitoring retention and storage
- ✅ ArgoCD sync policies
- ✅ Helm chart versions
- ✅ Resource requests and limits

---

## Configuration Files

### [.kube-linter.yaml](../../.kube-linter.yaml)
Configuration for kube-linter security checks. Excludes checks that may not apply to homelab environments while maintaining security best practices.

### [.markdownlint.json](../../.markdownlint.json)
Markdown linting rules for consistent documentation formatting.

### [.markdown-link-check.json](../../.markdown-link-check.json)
Configuration for link checking, including ignoring private IP ranges.

### [.gitleaks.toml](../../.gitleaks.toml)
Secret scanning rules with allowlists for encrypted SealedSecrets and public certificates.

---

## Workflow Status Badges

Add these badges to your main README.md to display pipeline status:

```markdown
![Validation](https://github.com/thiemotorres/homelab/actions/workflows/validate.yml/badge.svg)
![Security](https://github.com/thiemotorres/homelab/actions/workflows/security.yml/badge.svg)
![Documentation](https://github.com/thiemotorres/homelab/actions/workflows/docs.yml/badge.svg)
```

---

## Running Workflows Manually

All workflows support manual triggering via `workflow_dispatch`:

1. Go to **Actions** tab in GitHub
2. Select the workflow you want to run
3. Click **Run workflow**
4. Choose the branch (default: main)
5. Click **Run workflow**

---

## Integration with GitOps

These pipelines are designed to work seamlessly with ArgoCD:

1. **Pull Request Flow**:
   ```
   Developer commits → PR opened → Validation runs → Security scans
   ↓ (if passed)
   PR merged to main → ArgoCD detects change → Sync to cluster
   ```

2. **Protection**:
   - Validation failures block PR merges
   - Security issues are flagged for review
   - Only valid manifests reach ArgoCD

3. **Fast Feedback**:
   - Validation runs in ~3-5 minutes
   - No cluster access required for validation
   - Developers get immediate feedback

---

## Troubleshooting

### Validation Failures

**YAML Syntax Errors**:
```bash
# Validate locally before committing
yamllint argocd-apps/ infrastructure/ apps/
```

**Kubernetes Manifest Errors**:
```bash
# Test manifests locally
kubectl apply --dry-run=client -f infrastructure/postgres/postgres.yaml
```

**Kustomize Build Failures**:
```bash
# Build and inspect locally
kustomize build infrastructure/
kustomize build apps/gork/
```

### Security Scan Failures

**Secret Detection**:
- Ensure all secrets are SealedSecrets
- Check that .gitignore excludes secret files
- Review Gitleaks output for false positives

**RBAC Warnings**:
- Avoid wildcards (*) in RBAC rules
- Follow least privilege principle
- Document why specific permissions are needed

**Resource Limits**:
- Add resource requests and limits to all deployments
- Reference [../docs/README.md](../../docs/README.md) for guidelines

### Documentation Failures

**Broken Links**:
- Fix or update outdated URLs
- Use relative paths for internal links
- Check that referenced files exist

**Spell Check Warnings**:
- Add technical terms to `.github/workflows/docs.yml` custom dictionary
- Fix genuine typos

---

## Best Practices

### Before Committing

1. **Validate locally**:
   ```bash
   # YAML syntax
   yamllint infrastructure/

   # Kubernetes manifests
   kubectl apply --dry-run=client -f infrastructure/

   # Kustomize builds
   kustomize build infrastructure/
   ```

2. **Check for secrets**:
   ```bash
   # Scan for accidental secrets
   gitleaks detect --source . --verbose
   ```

3. **Lint documentation**:
   ```bash
   # Check markdown files
   markdownlint docs/
   ```

### Pull Request Guidelines

1. **Keep PRs focused**: One feature/fix per PR
2. **Wait for checks**: Don't merge until all checks pass
3. **Review security warnings**: Address flagged security issues
4. **Update documentation**: Keep docs in sync with code changes
5. **Test locally first**: Run validations before pushing

### Continuous Improvement

1. **Monitor weekly reports**: Review scheduled health checks
2. **Update dependencies**: Keep Helm charts up to date
3. **Review security findings**: Address recommendations
4. **Improve documentation**: Fix broken links and add missing docs

---

## CI/CD Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     GitHub Repository                        │
│                     (Source of Truth)                        │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   │ Push / PR
                   ▼
┌─────────────────────────────────────────────────────────────┐
│                   GitHub Actions CI/CD                       │
├─────────────────┬──────────────┬──────────────┬─────────────┤
│   Validation    │   Security   │     Docs     │  Scheduled  │
│                 │              │              │   Checks    │
│ • YAML Lint     │ • Gitleaks   │ • Markdown   │ • Backups   │
│ • K8s Validate  │ • kube-lint  │ • Links      │ • Certs     │
│ • Kustomize     │ • RBAC       │ • Examples   │ • Deps      │
│ • Helm          │ • Kubesec    │ • Spelling   │ • Resources │
└─────────────────┴──────────────┴──────────────┴─────────────┘
                   │
                   │ ✅ All checks passed
                   ▼
┌─────────────────────────────────────────────────────────────┐
│                    Merge to Main Branch                      │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   │ ArgoCD polls every 3 minutes
                   ▼
┌─────────────────────────────────────────────────────────────┐
│                         ArgoCD                               │
│                    (GitOps Operator)                         │
│                                                              │
│  • Detects changes                                          │
│  • Automated sync (prune + self-heal)                       │
│  • Applies manifests to cluster                             │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   │ kubectl apply
                   ▼
┌─────────────────────────────────────────────────────────────┐
│                   K3s Kubernetes Cluster                     │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │Infrastructure│  │  Monitoring  │  │     Apps     │     │
│  │              │  │              │  │              │     │
│  │ • Traefik    │  │ • Prometheus │  │ • Gork       │     │
│  │ • PostgreSQL │  │ • Grafana    │  │ • n8n        │     │
│  │ • cert-mgr   │  │ • Alerts     │  │ • Ext Svc    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

---

## Future Enhancements

Consider adding these workflows in the future:

1. **Image Scanning**: Scan container images for vulnerabilities (Trivy)
2. **Integration Tests**: Test application deployments in ephemeral environments
3. **Performance Testing**: Load testing for applications
4. **Compliance Scanning**: OPA/Gatekeeper policy validation
5. **Automated PRs**: Dependabot for Helm chart updates
6. **Slack/Discord Notifications**: Workflow status notifications
7. **Drift Detection**: Compare live cluster state with Git

---

## Support

For issues or questions:
- Review the [main documentation](../../docs/README.md)
- Check workflow run logs in GitHub Actions
- Open an issue in the repository

---

**Last Updated**: 2025-10-26
