# Guide: Exposing External Services

This guide shows you how to expose external services (running outside Kubernetes) through the cluster's Traefik ingress with automatic TLS certificates.

> **📚 Related Documentation**: [Ingress and Routing](../INGRESS-AND-ROUTING.md#example-4-external-service-proxy-router)

## What This Guide Covers

By the end of this guide, you'll be able to:
- Expose any device on your network through a clean HTTPS URL
- Automatically get Let's Encrypt TLS certificates
- Add authentication and access control
- Monitor access through centralized logs

## Prerequisites

- K3s cluster with ArgoCD deployed
- Traefik ingress controller running
- cert-manager configured with Cloudflare DNS
- Device accessible on your local network

## Use Cases

Perfect for exposing:
- **Home Router** - Clean URL instead of remembering 192.168.x.x
- **NAS** - Easy access to file server
- **Pi-hole** - DNS admin interface
- **Printer** - Web interface for network printer
- **IoT Devices** - Cameras, smart home hubs, etc.

## How It Works

```
External Device (192.168.92.100:8080)
         ↓
Kubernetes Service (ClusterIP)
         ↓
Kubernetes Endpoints (manual, pointing to external IP)
         ↓
Traefik IngressRoute (routing rules)
         ↓
cert-manager Certificate (Let's Encrypt)
         ↓
TLS Secret (certificate stored here)
         ↓
https://mydevice.feto.dev (public access!)
```

**Key insight**: We create a Kubernetes Service without a selector and manually define Endpoints that point to the external IP address.

---

## Quick Start: Expose a NAS

Let's expose a Synology NAS running at `192.168.92.100:5000` as `https://nas.feto.dev`.

### Step 1: Create the Configuration

Create `apps/external-services/nas-service.yaml`:

```yaml
---
# Service (without selector - we'll manually define endpoints)
apiVersion: v1
kind: Service
metadata:
  name: nas-service
  namespace: external-services
  labels:
    app: nas
spec:
  ports:
  - port: 5000
    targetPort: 5000
    protocol: TCP
    name: http

---
# Endpoints (manually point to external IP)
apiVersion: v1
kind: Endpoints
metadata:
  name: nas-service  # Must match Service name
  namespace: external-services
subsets:
- addresses:
  - ip: 192.168.92.100  # Your NAS IP
  ports:
  - port: 5000          # Your NAS port
    protocol: TCP
    name: http

---
# Certificate (Let's Encrypt via DNS-01)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: nas-cert
  namespace: external-services
spec:
  secretName: nas-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - nas.feto.dev

---
# IngressRoute (Traefik routing)
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nas
  namespace: external-services
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`nas.feto.dev`)
      kind: Rule
      services:
        - name: nas-service
          port: 5000
  tls:
    secretName: nas-tls
```

### Step 2: Add to Kustomization

Edit `apps/external-services/kustomization.yaml`:

```yaml
resources:
  - namespace.yaml
  - router-service.yaml
  - nas-service.yaml  # Add this line
```

### Step 3: Configure DNS

In Cloudflare, add an A record:

```
nas.feto.dev → <your-cluster-external-ip>
```

Or use a wildcard:
```
*.feto.dev → <your-cluster-external-ip>
```

### Step 4: Deploy

```bash
git add apps/external-services/nas-service.yaml
git add apps/external-services/kustomization.yaml
git commit -m "Add NAS external service proxy"
git push
```

### Step 5: Wait for Certificate

ArgoCD will deploy within ~3 minutes, then cert-manager will request the certificate:

```bash
# Watch certificate progress
kubectl get certificate -n external-services -w

# Should show Ready=True after 1-2 minutes
```

### Step 6: Test Access

```bash
curl -I https://nas.feto.dev
# Should return 200 OK with valid TLS certificate
```

Open in browser: **https://nas.feto.dev** ✅

---

## Advanced Scenarios

### HTTPS Backend (Router Example)

When the external service already uses HTTPS (like a router admin interface):

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: router-service
  namespace: external-services
spec:
  ports:
  - port: 440  # Can use any port in k8s
    targetPort: 440
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  name: router-service
  namespace: external-services
subsets:
- addresses:
  - ip: 192.168.92.1  # Router typically at gateway
  ports:
  - port: 440  # Router's HTTPS port
    protocol: TCP

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: router-cert
  namespace: external-services
spec:
  secretName: router-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - router.feto.dev

---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: router-headers
  namespace: external-services
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: "https"
    customResponseHeaders:
      X-Frame-Options: "SAMEORIGIN"
      X-Content-Type-Options: "nosniff"

---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: router
  namespace: external-services
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`router.feto.dev`)
      kind: Rule
      services:
        - name: router-service
          port: 440
      middlewares:
        - name: router-headers
  tls:
    secretName: router-tls
```

**Note**: Traefik terminates TLS at the ingress, then connects to the router's HTTPS backend. The router will show a certificate warning (self-signed), but users only see the valid Let's Encrypt cert.

---

## Security: IP Whitelisting

Restrict access to local network only:

```yaml
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: local-only
  namespace: external-services
spec:
  ipWhiteList:
    sourceRange:
      - "192.168.0.0/16"   # All local network IPs
      - "10.0.0.0/8"       # Include if needed
      - "172.16.0.0/12"    # Docker networks

---
# Add to IngressRoute
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nas
  namespace: external-services
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`nas.feto.dev`)
      kind: Rule
      services:
        - name: nas-service
          port: 5000
      middlewares:
        - name: local-only  # Apply whitelist
  tls:
    secretName: nas-tls
```

Now `nas.feto.dev` only works from local network IPs.

---

## Security: Basic Authentication

Add password protection:

### Step 1: Create htpasswd Secret

```bash
# Install htpasswd (if needed)
sudo apt-get install apache2-utils

# Generate password file
htpasswd -nb admin 'your-secure-password' > auth

# Create Kubernetes secret
kubectl create secret generic nas-auth \
  --from-file=auth \
  --namespace=external-services
```

### Step 2: Create Middleware

```yaml
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: nas-auth
  namespace: external-services
spec:
  basicAuth:
    secret: nas-auth
```

### Step 3: Apply to IngressRoute

```yaml
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nas
  namespace: external-services
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`nas.feto.dev`)
      kind: Rule
      services:
        - name: nas-service
          port: 5000
      middlewares:
        - name: nas-auth  # Require login
  tls:
    secretName: nas-tls
```

Now accessing `https://nas.feto.dev` prompts for username/password.

---

## Combining Security Layers

Stack multiple middlewares:

```yaml
spec:
  routes:
    - match: Host(`nas.feto.dev`)
      kind: Rule
      services:
        - name: nas-service
          port: 5000
      middlewares:
        - name: local-only    # First: check IP
        - name: nas-auth      # Then: require login
        - name: security-headers  # Finally: add headers
```

Middlewares are applied in order.

---

## Troubleshooting

### Certificate Stuck on Pending

**Check certificate status:**
```bash
kubectl describe certificate nas-cert -n external-services
```

**Common issues:**
- Cloudflare API token expired
- DNS zone mismatch (using wrong token)
- DNS propagation delay (usually resolves in 1-2 min)

**Solution:**
```bash
# Delete and recreate certificate
kubectl delete certificate nas-cert -n external-services
kubectl apply -f apps/external-services/nas-service.yaml
```

### 502 Bad Gateway

**Check if service is reachable:**
```bash
# Test from within cluster
kubectl run debug -it --rm --image=nicolaka/netshoot -- \
  curl http://192.168.92.100:5000
```

**Common causes:**
- External device is down
- Wrong IP address in Endpoints
- Wrong port number
- Firewall blocking cluster → device traffic

**Fix:**
```bash
# Verify endpoints
kubectl get endpoints nas-service -n external-services

# Should show:
# ENDPOINTS        192.168.92.100:5000
```

### Cannot Access from Browser

**Check DNS:**
```bash
# Resolve domain
dig nas.feto.dev

# Should return your cluster IP
```

**Check Traefik:**
```bash
# View Traefik logs
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik | grep nas

# Check IngressRoute
kubectl get ingressroute -n external-services
kubectl describe ingressroute nas -n external-services
```

### Certificate Valid but Browser Shows Warning

**Cause**: Browser cached old certificate or accessing via IP instead of domain

**Solution:**
- Clear browser cache
- Access via domain name (not IP)
- Check certificate: `openssl s_client -connect nas.feto.dev:443 -servername nas.feto.dev`

---

## Template for Quick Setup

Save as `apps/external-services/TEMPLATE.yaml`:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: CHANGEME-service
  namespace: external-services
spec:
  ports:
  - port: CHANGEPORT
    targetPort: CHANGEPORT
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  name: CHANGEME-service
  namespace: external-services
subsets:
- addresses:
  - ip: CHANGEIP
  ports:
  - port: CHANGEPORT
    protocol: TCP

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: CHANGEME-cert
  namespace: external-services
spec:
  secretName: CHANGEME-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - CHANGEME.feto.dev

---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: CHANGEME
  namespace: external-services
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`CHANGEME.feto.dev`)
      kind: Rule
      services:
        - name: CHANGEME-service
          port: CHANGEPORT
  tls:
    secretName: CHANGEME-tls
```

**Usage:**
```bash
cp TEMPLATE.yaml printer-service.yaml
# Replace CHANGEME with "printer"
# Replace CHANGEIP with device IP
# Replace CHANGEPORT with device port
```

---

## Real-World Examples

### Pi-hole DNS Server

```yaml
# Pi-hole at 192.168.92.50:80
subsets:
- addresses:
  - ip: 192.168.92.50
  ports:
  - port: 80

# Access: https://pihole.feto.dev
dnsNames:
  - pihole.feto.dev
```

### Security Camera

```yaml
# Camera at 192.168.92.201:554 (RTSP)
# Often cameras have web UI on port 80
subsets:
- addresses:
  - ip: 192.168.92.201
  ports:
  - port: 80

# Access: https://camera.feto.dev
dnsNames:
  - camera.feto.dev
```

### Network Printer

```yaml
# HP Printer at 192.168.92.150:80
subsets:
- addresses:
  - ip: 192.168.92.150
  ports:
  - port: 80

# Access: https://printer.feto.dev
dnsNames:
  - printer.feto.dev
```

---

## Best Practices

### 1. Use Descriptive Names

✅ `nas-service`, `router-service`, `printer-service`
❌ `service1`, `external-svc`, `test`

### 2. Document Device IPs

Add comments:
```yaml
subsets:
- addresses:
  - ip: 192.168.92.100  # Synology NAS - main storage
  ports:
  - port: 5000
```

### 3. Use Separate Secrets for Auth

Don't reuse the same htpasswd secret:
```yaml
# nas-auth, router-auth, printer-auth
# Different passwords per service
```

### 4. Test Certificate Renewal

Certificates renew automatically, but test:
```bash
# Check expiry
kubectl get certificate -n external-services

# Force renewal (if needed)
kubectl delete secret nas-tls -n external-services
# cert-manager recreates automatically
```

### 5. Monitor Access

Check Traefik metrics in Grafana:
- Request rates per service
- Response times
- Error rates

---

## Benefits Recap

✅ **Clean URLs** - `nas.feto.dev` instead of `http://192.168.92.100:5000`
✅ **Automatic HTTPS** - Let's Encrypt certificates for all services
✅ **Centralized Access** - All devices through one ingress point
✅ **Easy Maintenance** - Change device IP in one place (Endpoints)
✅ **Security** - Add auth, IP whitelisting, headers as needed
✅ **Monitoring** - All access logged and metrics collected
✅ **GitOps** - Configuration in Git, deployed automatically

---

## Related Guides

- [RBAC Configuration](../RBAC.md) - Secure ServiceAccounts
- [Certificate Management](../CERTIFICATE-MANAGEMENT.md) - TLS deep dive
- [Ingress and Routing](../INGRESS-AND-ROUTING.md) - Traefik configuration

---

**Questions or Issues?** See the [main documentation](../README.md) or check [troubleshooting section](#troubleshooting).
