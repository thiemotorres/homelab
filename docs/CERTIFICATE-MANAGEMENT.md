# TLS Certificate Management

This document describes the automated TLS certificate management using cert-manager and Let's Encrypt.

## Overview

The homelab uses **cert-manager** to automatically obtain and renew TLS certificates from **Let's Encrypt**, providing:
- Automated certificate issuance via DNS-01 validation
- Automatic renewal (30 days before expiry)
- Separate certificates per service
- Support for two domains (feto.dev, thiemo.click)
- Zero-downtime certificate rotation

## Architecture

```
┌──────────────────┐
│  cert-manager    │
│  (Controller)    │
└────────┬─────────┘
         │
         │ 1. Watch Certificate resources
         │
         ▼
┌──────────────────┐
│   Certificate    │
│   (CRD)          │
└────────┬─────────┘
         │
         │ 2. Create ACME Order
         │
         ▼
┌──────────────────────────┐
│   Let's Encrypt API      │
│   (ACME Server)          │
└────────┬─────────────────┘
         │
         │ 3. DNS-01 Challenge
         │
         ▼
┌──────────────────────────┐      ┌──────────────────┐
│   Cloudflare DNS API     │◄─────┤  API Token       │
│   (TXT Record)           │      │  (Secret)        │
└────────┬─────────────────┘      └──────────────────┘
         │
         │ 4. Validation Success
         │
         ▼
┌──────────────────────────┐
│   Kubernetes Secret      │
│   (tls.crt + tls.key)    │
└────────┬─────────────────┘
         │
         │ 5. Used by Traefik
         │
         ▼
┌──────────────────────────┐
│   IngressRoute (TLS)     │
└──────────────────────────┘
```

## Components

### cert-manager Installation

**Location**: Deployed via ArgoCD
**File**: `argocd-apps/cert-manager.yaml`
**Helm Chart**: jetstack/cert-manager v1.16.2

**Configuration:**
```yaml
installCRDs: true  # Installs Certificate, ClusterIssuer CRDs

# DNS server configuration for DNS-01 challenges
extraArgs:
  - --dns01-recursive-nameservers-only=true
  - --dns01-recursive-nameservers=1.1.1.1:53,8.8.8.8:53
```

### ClusterIssuers

**Location**: `infrastructure/cert-manager/`

Two ClusterIssuers configured:

1. **letsencrypt-prod** - Production certificates
2. **letsencrypt-staging** - Testing (higher rate limits)

---

## ClusterIssuer Configuration

### Production Issuer (feto.dev)

**File**: `infrastructure/cert-manager/clusterissuer-prod.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: info@thiemo-torres.de  # Let's Encrypt notifications
    privateKeySecretRef:
      name: letsencrypt-prod-key  # ACME account key

    solvers:
    # Solver for feto.dev domain
    - selector:
        dnsZones:
          - "feto.dev"
      dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token-feto
            key: api-token

    # Solver for thiemo.click domain
    - selector:
        dnsZones:
          - "thiemo.click"
      dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token-thiemo
            key: api-token
```

**Key Points:**
- Uses DNS-01 validation (creates TXT records)
- Separate Cloudflare API tokens per domain
- Stored as Sealed Secrets for security

### Staging Issuer

**File**: `infrastructure/cert-manager/clusterissuer-staging.yaml`

```yaml
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # ... same configuration as production
```

**When to use staging:**
- Testing new certificate configurations
- Avoiding rate limits during development
- Let's Encrypt production has strict rate limits (50 certs/week per domain)

---

## Certificate Resources

Each service has a Certificate resource defining its TLS certificate.

### Example: Gork Certificate

**File**: `apps/gork/certificate.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: gork-cert
  namespace: gork
spec:
  secretName: gork-tls           # Where cert is stored
  issuerRef:
    name: letsencrypt-prod       # Which issuer to use
    kind: ClusterIssuer
  dnsNames:
    - gork.thiemo.click          # Domain name(s)
  # Optional: renewal before expiry
  # renewBefore: 720h            # Renew 30 days before expiry (default)
```

### Certificate Lifecycle

```
1. Certificate created
   ↓
2. cert-manager detects new Certificate
   ↓
3. Creates CertificateRequest
   ↓
4. Contacts Let's Encrypt ACME server
   ↓
5. Receives DNS-01 challenge
   ↓
6. Creates TXT record via Cloudflare API
   _acme-challenge.gork.thiemo.click → "random-token"
   ↓
7. Let's Encrypt validates TXT record
   ↓
8. Certificate issued (valid 90 days)
   ↓
9. Stored in Secret (gork-tls)
   ↓
10. Traefik reads secret for TLS termination
```

---

## All Configured Certificates

| Service | Domain | Certificate Name | Secret Name | Namespace |
|---------|--------|-----------------|-------------|-----------|
| Gork | gork.thiemo.click | gork-cert | gork-tls | gork |
| n8n | n8n.thiemo.click | n8n-cert | n8n-tls | n8n |
| ArgoCD | argocd.feto.dev | argocd-cert | argocd-tls | argocd |
| Grafana | grafana.feto.dev | grafana-cert | grafana-tls | monitoring |
| Router | router.feto.dev | router-cert | router-tls | external-services |

---

## Cloudflare API Token Configuration

### Required Permissions

Cloudflare API tokens need:
- **Zone:DNS:Edit** - Create/delete TXT records
- **Zone:Zone:Read** - Read zone information

### Token Scope

**feto.dev token** (`cloudflare-api-token-feto`):
- Zone: feto.dev only
- Permissions: DNS Edit, Zone Read

**thiemo.click token** (`cloudflare-api-token-thiemo`):
- Zone: thiemo.click only
- Permissions: DNS Edit, Zone Read

### Security Benefits

**Separate tokens per domain:**
- Compromise of one token doesn't affect other domain
- Principle of least privilege
- Easier to rotate individual tokens

### Creating API Tokens

1. **Login to Cloudflare Dashboard**
2. **Go to**: My Profile → API Tokens
3. **Create Token** with template "Edit zone DNS"
4. **Configure**:
   - Permissions: Zone > DNS > Edit + Zone > Zone > Read
   - Zone Resources: Specific zone (e.g., feto.dev)
5. **Copy token** (shown only once!)
6. **Create Sealed Secret**:

```bash
# Create secret manifest
kubectl create secret generic cloudflare-api-token-feto \
  --from-literal=api-token='your-token-here' \
  --namespace=cert-manager \
  --dry-run=client -o yaml > /tmp/secret.yaml

# Seal it
kubeseal --format=yaml --cert=pub-cert.pem \
  < /tmp/secret.yaml \
  > infrastructure/cert-manager/sealed-secrets/cloudflare-api-token-feto-sealed.yaml

# Clean up
rm /tmp/secret.yaml
```

---

## DNS-01 vs HTTP-01 Validation

### Why DNS-01?

**Advantages:**
✅ Works with any domain (no need for public IP)
✅ Works behind firewalls/NAT
✅ Can issue wildcard certificates (`*.example.com`)
✅ No need to expose port 80

**Disadvantages:**
❌ Requires DNS provider API access
❌ Slightly slower validation
❌ More complex setup

### HTTP-01 (Not Used)

**Would require:**
- Port 80 accessible from internet
- Special ACME validation endpoint
- Can't do wildcards

**For homelab, DNS-01 is better** because:
- Many homelabs are behind CGNAT (no public IP)
- Router may not forward port 80
- More flexible for future wildcard certs

---

## Certificate Renewal

### Automatic Renewal

cert-manager automatically renews certificates:

**Default behavior:**
- Checks certificates daily
- Renews when < 30 days until expiry
- Zero-downtime renewal (new cert in secret)

**Can be customized:**
```yaml
spec:
  renewBefore: 720h  # 30 days (default)
  # or
  renewBefore: 1440h  # 60 days (more buffer)
```

### Monitoring Renewal

```bash
# Check certificate expiry
kubectl get certificate -A

# Detailed certificate info
kubectl describe certificate gork-cert -n gork

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# View certificate details from secret
kubectl get secret gork-tls -n gork -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | \
  openssl x509 -text -noout
```

---

## Troubleshooting

### Certificate Not Ready

**Symptoms**: Certificate shows `Ready=False`

**Check status:**
```bash
kubectl get certificate gork-cert -n gork
kubectl describe certificate gork-cert -n gork
```

**Common issues:**

#### 1. DNS-01 Challenge Failed

```bash
# Check CertificateRequest
kubectl get certificaterequest -n gork
kubectl describe certificaterequest <name> -n gork

# Check Challenge
kubectl get challenge -n gork
kubectl describe challenge <name> -n gork
```

**Possible causes:**
- Invalid Cloudflare API token
- Token doesn't have DNS Edit permission
- Token scoped to wrong zone
- DNS propagation delay

**Solution:**
```bash
# Verify API token in secret
kubectl get secret cloudflare-api-token-feto -n cert-manager -o yaml

# Test token manually (outside cluster)
curl -X GET "https://api.cloudflare.com/client/v4/zones" \
  -H "Authorization: Bearer YOUR-TOKEN-HERE"

# Delete and recreate certificate to retry
kubectl delete certificate gork-cert -n gork
kubectl apply -f apps/gork/certificate.yaml
```

#### 2. Rate Limit Exceeded

**Let's Encrypt rate limits:**
- 50 certificates per registered domain per week
- 5 duplicate certificates per week

**Solution:**
- Wait for rate limit window to reset
- Use staging issuer for testing
- Be careful with certificate recreation

#### 3. Wrong Issuer Reference

```yaml
# Check issuerRef matches ClusterIssuer name
spec:
  issuerRef:
    name: letsencrypt-prod  # Must match ClusterIssuer
    kind: ClusterIssuer     # Not "Issuer"
```

### Certificate Expired

**Should never happen** (auto-renewal), but if it does:

```bash
# Check certificate status
kubectl get certificate -A

# Force renewal by deleting secret
kubectl delete secret gork-tls -n gork

# cert-manager will automatically recreate it
```

### cert-manager Not Working

```bash
# Check cert-manager pods
kubectl get pods -n cert-manager

# Check logs
kubectl logs -n cert-manager deployment/cert-manager

# Check webhook
kubectl get validatingwebhookconfiguration
kubectl get mutatingwebhookconfiguration

# Test webhook connectivity
kubectl run test-cert --rm -i --tty --image=curlimages/curl -- \
  curl -v https://cert-manager-webhook.cert-manager.svc:443/validate
```

---

## Adding New Certificates

### Step 1: Create Certificate Resource

```yaml
# apps/myapp/certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: myapp
spec:
  secretName: myapp-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - myapp.thiemo.click  # or feto.dev
```

### Step 2: Add to Kustomization

```yaml
# apps/myapp/kustomization.yaml
resources:
  - certificate.yaml
  - ...
```

### Step 3: Reference in IngressRoute

```yaml
# apps/myapp/ingressroute.yaml
spec:
  tls:
    secretName: myapp-tls
```

### Step 4: Wait for Issuance

```bash
# Monitor certificate
kubectl get certificate myapp-cert -n myapp -w

# Should show Ready=True after ~1-2 minutes
```

---

## Wildcard Certificates (Optional)

Currently, each service has its own certificate. Alternatively, use wildcards:

### Wildcard Certificate Example

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-thiemo-click
  namespace: cert-manager
spec:
  secretName: wildcard-thiemo-click-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - "*.thiemo.click"
    - "thiemo.click"  # Include apex domain too
```

**Advantages:**
- One certificate for all subdomains
- Fewer Let's Encrypt requests
- Easier management

**Disadvantages:**
- Secret must be copied to each namespace
- Less granular control
- If compromised, affects all subdomains

**Recommendation**: Stick with per-service certificates for homelab (current setup is better).

---

## Certificate Rotation

### Automatic Rotation

When cert-manager renews a certificate:

1. New certificate issued
2. Secret updated with new cert
3. Traefik automatically detects change
4. New connections use new cert
5. Zero downtime!

### Manual Rotation (Force Renewal)

```bash
# Delete the secret (cert-manager will recreate)
kubectl delete secret gork-tls -n gork

# Or use cmctl (cert-manager CLI)
cmctl renew gork-cert -n gork

# Or delete and recreate certificate
kubectl delete certificate gork-cert -n gork
kubectl apply -f apps/gork/certificate.yaml
```

---

## Monitoring

### Certificate Expiry Alerts

**Recommended**: Add Prometheus alerts for certificate expiry

```yaml
# Example PrometheusRule (not currently configured)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: certificate-expiry
  namespace: monitoring
spec:
  groups:
  - name: certificates
    rules:
    - alert: CertificateExpiringSoon
      expr: |
        certmanager_certificate_expiration_timestamp_seconds - time() < (21 * 24 * 3600)
      for: 1h
      annotations:
        summary: "Certificate {{ $labels.name }} expiring in < 21 days"
      labels:
        severity: warning
```

### Check Certificate Expiry

```bash
# Via kubectl
kubectl get certificate -A -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name): \(.status.notAfter)"'

# Via openssl (from secret)
kubectl get secret gork-tls -n gork -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | \
  openssl x509 -noout -dates
```

---

## Security Best Practices

### 1. Protect API Tokens

✅ Stored as Sealed Secrets (encrypted at rest)
✅ Separate tokens per domain
✅ Minimum required permissions

### 2. Rotate API Tokens

**Recommended frequency**: Annually

**Steps:**
1. Create new Cloudflare API token
2. Create new Sealed Secret
3. Apply to cluster
4. Verify certificates still renew
5. Delete old token from Cloudflare

### 3. Monitor Certificate Health

- Set up alerts for expiry
- Check cert-manager logs regularly
- Test renewal process periodically

### 4. Use Production Issuer

✅ All production services use `letsencrypt-prod`
❌ Don't use staging certs in production (browser warnings)

---

## Backup and Disaster Recovery

### Certificate Secrets

**Certificates are stored in Secrets**:
- Can be backed up with standard secret backup
- cert-manager will recreate if lost
- No manual backup needed (cert-manager is source of truth)

### If cert-manager is Lost

```bash
# Reinstall cert-manager via ArgoCD
# ClusterIssuers will be recreated
# Certificates will be recreated
# New certificates will be issued automatically

# No manual intervention needed!
```

### If Let's Encrypt Account Key is Lost

```bash
# Delete the account key secret
kubectl delete secret letsencrypt-prod-key -n cert-manager

# cert-manager will create new account
# All certificates will be re-issued
# May hit rate limits if many certs
```

---

## Advanced Configurations

### Custom Certificate Duration

```yaml
spec:
  duration: 2160h    # 90 days (Let's Encrypt default)
  renewBefore: 360h  # Renew 15 days before expiry
```

### Multiple DNS Names

```yaml
spec:
  dnsNames:
    - gork.thiemo.click
    - www.gork.thiemo.click
    - api.gork.thiemo.click
```

### IP SANs (Subject Alternative Names)

```yaml
spec:
  ipAddresses:
    - "192.168.1.100"
  # Useful for services accessed by IP
```

### Custom Certificate Usages

```yaml
spec:
  usages:
    - digital signature
    - key encipherment
    - server auth
    # Can add: client auth, code signing, etc.
```

---

## Comparison: cert-manager vs Manual Certificates

### Manual Process (Without cert-manager)

❌ Generate CSR manually
❌ Submit to Let's Encrypt manually
❌ Complete DNS validation manually
❌ Download certificate manually
❌ Create Kubernetes secret manually
❌ Remember to renew every 90 days
❌ Update secret manually on renewal

### Automated (With cert-manager)

✅ Create one Certificate YAML
✅ Everything else automatic
✅ Auto-renewal
✅ Zero-downtime rotation
✅ Prometheus metrics
✅ GitOps-friendly

---

## References

- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Cloudflare API Documentation](https://developers.cloudflare.com/api/)
- [ACME Protocol (RFC 8555)](https://datatracker.ietf.org/doc/html/rfc8555)

---

## Changelog

- **2025-10-26**: Documented existing cert-manager configuration
  - Let's Encrypt with DNS-01 validation
  - Dual-domain support (feto.dev, thiemo.click)
  - Automatic renewal configuration
  - Cloudflare API token setup
