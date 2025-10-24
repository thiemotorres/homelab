# Homelab

A Kubernetes-based homelab running on K3s with GitOps practices using ArgoCD.

## Overview

This repository contains the infrastructure and application configurations for a personal homelab cluster. The setup uses K3s (lightweight Kubernetes), ArgoCD for GitOps-based deployments, and automated TLS certificate management via cert-manager with Let's Encrypt.

## Services

### Infrastructure

- **ArgoCD** - GitOps continuous delivery tool for Kubernetes
- **Traefik** - Ingress controller and reverse proxy
- **cert-manager** - Automated TLS certificate management using Let's Encrypt with Cloudflare DNS-01 challenges
- **Sealed Secrets** - Encrypted secrets management for GitOps workflows

### Domains

- **feto.dev** - Internal services (local network access only)
- **thiemo.click** - Public-facing services (internet accessible)

Both domains use Cloudflare DNS for automatic Let's Encrypt certificate provisioning.

### Applications

- **whoami** - Demo service on feto.dev (local)
- **hello-world** - Demo service on thiemo.click (public)

## Quick Start

### Prerequisites

- K3s cluster
- kubectl configured
- Helm (optional)
- Cloudflare API tokens for DNS management

### Installation

1. Install ArgoCD:
```bash
kubectl create namespace argocd
kubectl apply -k bootstrap/argocd/
```

2. Access ArgoCD UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get initial password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

3. Create Cloudflare secrets for cert-manager:
```bash
kubectl create namespace cert-manager

kubectl create secret generic cloudflare-api-token-feto \
  --namespace cert-manager \
  --from-literal=api-token='YOUR_FETO_DEV_TOKEN'

kubectl create secret generic cloudflare-api-token-thiemo \
  --namespace cert-manager \
  --from-literal=api-token='YOUR_THIEMO_CLICK_TOKEN'
```

4. Deploy all applications:
```bash
kubectl apply -f argocd-apps/root-app.yaml
```

ArgoCD will automatically deploy infrastructure and applications.

## Repository Structure

```
.
├── bootstrap/           # ArgoCD installation
├── argocd-apps/         # ArgoCD Application definitions (App of Apps pattern)
├── infrastructure/      # Core infrastructure components
│   ├── namespaces/
│   ├── sealed-secrets/
│   ├── traefik/
│   └── cert-manager/
├── apps/                # User applications
└── secrets/             # Secrets management documentation
```

## Useful Commands

### Check certificate status
```bash
kubectl get certificate -A
kubectl describe certificate <cert-name> -n <namespace>
```

### View ArgoCD applications
```bash
kubectl get applications -n argocd
```

### Debug cert-manager
```bash
kubectl logs -n cert-manager deployment/cert-manager -f
```

## Documentation

- See [secrets/README.md](secrets/README.md) for secrets management details
- See [secrets/SEALED-SECRETS-WORKFLOW.md](secrets/SEALED-SECRETS-WORKFLOW.md) for Sealed Secrets usage

## Contact

info@thiemo-torres.de
