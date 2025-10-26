# Ingress and Routing Configuration

This document describes how external traffic reaches your applications through Traefik ingress controller.

## Overview

The homelab uses **Traefik** as the ingress controller and reverse proxy, providing:
- HTTPS termination with automatic TLS certificates
- HTTP to HTTPS redirection
- Custom security headers
- Load balancing across multiple nodes
- WebSocket support (for n8n)

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Internet                         │
└────────────────────┬────────────────────────────────┘
                     │
                     │ HTTPS (443)
                     ▼
          ┌──────────────────────┐
          │  Home Router/Firewall │
          │  Port Forward 443     │
          └──────────┬────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │   K3s LoadBalancer    │
          │   (Any Node IP:443)   │
          └──────────┬────────────┘
                     │
       ┌─────────────┼─────────────┐
       │             │             │
       ▼             ▼             ▼
┌───────────┐ ┌───────────┐ ┌───────────┐
│  Traefik  │ │  Traefik  │ │  Traefik  │
│  (Node 1) │ │  (Node 2) │ │  (Node 3) │
└─────┬─────┘ └─────┬─────┘ └─────┬─────┘
      │             │             │
      └─────────────┼─────────────┘
                    │
       ┌────────────┼────────────┐
       │            │            │
       ▼            ▼            ▼
┌───────────┐ ┌─────────┐ ┌─────────┐
│   Gork    │ │   n8n   │ │ Grafana │
│ (Backend) │ │         │ │         │
└───────────┘ └─────────┘ └─────────┘
```

## Components

### Traefik Deployment

**Location**: Deployed via ArgoCD Application
**File**: `argocd-apps/traefik.yaml`
**Type**: DaemonSet (runs on every node)

**Key Configuration:**
```yaml
deployment:
  kind: DaemonSet  # Ensures Traefik runs on all nodes

ports:
  web:
    port: 80
    exposedPort: 80
  websecure:
    port: 443
    exposedPort: 443
    tls:
      enabled: true

service:
  type: LoadBalancer  # K3s built-in LoadBalancer
```

**Why DaemonSet?**
- High availability (runs on all nodes)
- No single point of failure
- Traffic can reach any node
- Ideal for K3s edge deployments

---

## Domain Strategy

### Two-Domain Setup

#### 1. feto.dev (Internal/Local Services)
**DNS**: Cloudflare
**Purpose**: Internal homelab services
**Services**:
- `argocd.feto.dev` - ArgoCD UI
- `grafana.feto.dev` - Grafana dashboards
- `router.feto.dev` - External service proxy (router admin)

#### 2. thiemo.click (Public Services)
**DNS**: Cloudflare
**Purpose**: Internet-facing applications
**Services**:
- `gork.thiemo.click` - Gork application
- `n8n.thiemo.click` - n8n automation platform

### Why Two Domains?

**Security Boundary:**
- Internal tools (ArgoCD, Grafana) on separate domain
- Public apps on different domain
- Easier to apply different security policies
- Clear separation of concerns

---

## IngressRoute Configuration

Traefik uses custom CRDs (IngressRoute) instead of standard Kubernetes Ingress.

### Basic IngressRoute Pattern

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: <service-name>
  namespace: <namespace>
spec:
  entryPoints:
    - websecure        # HTTPS (443)
  routes:
    - match: Host(`<domain>`)
      kind: Rule
      services:
        - name: <service-name>
          port: <port>
  tls:
    secretName: <tls-secret-name>
```

---

## Service Examples

### Example 1: Gork (Simple HTTPS)

**File**: `apps/gork/ingressroute.yaml`

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: gork
  namespace: gork
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`gork.thiemo.click`)
      kind: Rule
      services:
        - name: gork-frontend-service
          port: 3000
  tls:
    secretName: gork-tls
```

**Flow:**
1. User visits `https://gork.thiemo.click`
2. Traefik matches Host header
3. Routes to `gork-frontend-service:3000`
4. TLS termination using cert from `gork-tls` secret

---

### Example 2: Grafana (HTTP Redirect + Custom Headers)

**File**: `infrastructure/monitoring/grafana-ingress.yaml`

```yaml
---
# HTTPS route
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: grafana
  namespace: monitoring
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`grafana.feto.dev`)
      kind: Rule
      services:
        - name: monitoring-kube-prometheus-grafana
          port: 80
  tls:
    secretName: grafana-tls

---
# HTTP to HTTPS redirect
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: grafana-redirect
  namespace: monitoring
spec:
  entryPoints:
    - web           # HTTP (80)
  routes:
    - match: Host(`grafana.feto.dev`)
      kind: Rule
      services:
        - name: monitoring-kube-prometheus-grafana
          port: 80
      middlewares:
        - name: redirect-https

---
# Redirect middleware
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: monitoring
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

**Flow:**
1. HTTP request → redirected to HTTPS
2. HTTPS request → routed to Grafana
3. Both use same backend service

---

### Example 3: ArgoCD (Custom Headers)

**File**: `bootstrap/argocd/ingressroute.yaml`

```yaml
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-server
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`argocd.feto.dev`)
      kind: Rule
      services:
        - name: argocd-server
          port: 80
      middlewares:
        - name: argocd-server-headers
  tls:
    secretName: argocd-tls

---
# Security headers middleware
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: argocd-server-headers
  namespace: argocd
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: "https"
    customResponseHeaders:
      X-Frame-Options: "SAMEORIGIN"
      X-Content-Type-Options: "nosniff"
```

**Headers Explained:**
- `X-Forwarded-Proto: https` - Tells ArgoCD it's behind HTTPS proxy
- `X-Frame-Options: SAMEORIGIN` - Prevents clickjacking
- `X-Content-Type-Options: nosniff` - Prevents MIME-sniffing attacks

---

### Example 4: External Service Proxy (Router)

**File**: `apps/external-services/router-service.yaml`

```yaml
---
# Service with manual endpoints (external IP)
apiVersion: v1
kind: Service
metadata:
  name: router-service
  namespace: external-services
spec:
  ports:
    - protocol: TCP
      port: 440
      targetPort: 440

---
# Manual endpoints pointing to external device
apiVersion: v1
kind: Endpoints
metadata:
  name: router-service
  namespace: external-services
subsets:
  - addresses:
      - ip: 192.168.92.1  # Router IP on LAN
    ports:
      - port: 440

---
# IngressRoute with TLS termination
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

**Use Case:**
- Access router admin panel via clean HTTPS URL
- TLS termination at Traefik
- Backend connection to router over plain HTTP/HTTPS

---

## Middleware Types

### 1. Redirect Scheme (HTTP → HTTPS)

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
spec:
  redirectScheme:
    scheme: https
    permanent: true  # HTTP 301 redirect
```

### 2. Headers

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: security-headers
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: "https"
    customResponseHeaders:
      X-Frame-Options: "DENY"
      X-Content-Type-Options: "nosniff"
      X-XSS-Protection: "1; mode=block"
      Referrer-Policy: "strict-origin-when-cross-origin"
```

### 3. Rate Limiting (Example - Not Currently Used)

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
spec:
  rateLimit:
    average: 100  # 100 requests per period
    period: 1s    # per second
    burst: 50     # allow burst of 50
```

### 4. IP Whitelist (Example - Not Currently Used)

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: ip-whitelist
spec:
  ipWhiteList:
    sourceRange:
      - "192.168.0.0/16"  # Local network only
      - "10.0.0.0/8"
```

---

## TLS Configuration

All services use TLS certificates from cert-manager (Let's Encrypt).

### Certificate Workflow

```
1. IngressRoute references TLS secret
   ↓
2. cert-manager Certificate resource exists
   ↓
3. cert-manager requests cert from Let's Encrypt
   ↓
4. DNS-01 validation via Cloudflare
   ↓
5. Certificate stored in Kubernetes Secret
   ↓
6. Traefik reads secret for TLS termination
```

See [CERTIFICATE-MANAGEMENT.md](CERTIFICATE-MANAGEMENT.md) for details.

---

## Service Discovery

Traefik automatically discovers services in Kubernetes:

### Automatic Discovery (Not Used)

```yaml
# Traefik can auto-discover via annotations
metadata:
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
```

### Explicit IngressRoute (Used in Homelab)

**Why explicit?**
- More control over routing
- Easier to read and understand
- Clear separation of concerns
- Better GitOps integration

---

## Load Balancing

### Traefik Load Balancing

When multiple pods exist for a service:

```yaml
services:
  - name: gork-backend-service
    port: 8080
    # Traefik automatically load balances to all endpoints
```

**Default algorithm**: Round-robin

**Can be customized:**
```yaml
services:
  - name: gork-backend-service
    port: 8080
    strategy: roundrobin  # or: wrr (weighted), leastconn
```

### Node-Level Load Balancing

K3s LoadBalancer service distributes traffic across nodes:
- Traffic can hit any node on port 443
- Traefik DaemonSet on that node handles it
- Built-in redundancy

---

## Monitoring and Debugging

### View Traefik Logs

```bash
# View logs from all Traefik pods
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik --tail=100 -f

# View logs from specific pod
kubectl logs -n kube-system traefik-xxxxx
```

### Check IngressRoutes

```bash
# List all IngressRoutes
kubectl get ingressroute -A

# Describe specific IngressRoute
kubectl describe ingressroute gork -n gork

# Check for errors
kubectl get ingressroute -A -o json | jq '.items[] | select(.status != null)'
```

### Test Routing

```bash
# Test from outside cluster
curl -v https://gork.thiemo.click

# Test HTTP redirect
curl -v http://grafana.feto.dev
# Should return 301 redirect to HTTPS

# Test with custom headers
curl -v -H "Host: gork.thiemo.click" https://192.168.92.42

# Check TLS certificate
openssl s_client -connect gork.thiemo.click:443 -servername gork.thiemo.click
```

### Traefik Dashboard (Optional)

Currently disabled for security. To enable:

```yaml
# In argocd-apps/traefik.yaml
ingressRoute:
  dashboard:
    enabled: true  # Change from false
```

Then access at: `http://<any-node-ip>:9000/dashboard/`

---

## Common Issues and Solutions

### 1. 404 Not Found

**Symptoms**: Traefik responds with 404

**Causes**:
- IngressRoute doesn't exist
- Host header doesn't match
- Service name or port incorrect

**Debug**:
```bash
# Check IngressRoute exists
kubectl get ingressroute -n <namespace>

# Check service exists
kubectl get svc -n <namespace>

# Check Traefik logs
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik | grep <domain>
```

### 2. 502 Bad Gateway

**Symptoms**: Traefik can't reach backend

**Causes**:
- Backend pods not running
- Service selector doesn't match pods
- Wrong port in IngressRoute

**Debug**:
```bash
# Check pods are running
kubectl get pods -n <namespace>

# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>

# Test backend directly
kubectl port-forward svc/<service-name> 8080:8080 -n <namespace>
curl localhost:8080
```

### 3. TLS Certificate Issues

**Symptoms**: Certificate warnings in browser

**Causes**:
- Certificate not ready
- Wrong secret name
- cert-manager issues

**Debug**:
```bash
# Check certificate status
kubectl get certificate -n <namespace>

# Describe certificate
kubectl describe certificate <cert-name> -n <namespace>

# Check secret exists
kubectl get secret <tls-secret-name> -n <namespace>
```

### 4. Redirect Loop

**Symptoms**: Browser shows "too many redirects"

**Causes**:
- Both HTTP and HTTPS routes redirecting
- Backend redirecting to HTTP

**Solution**:
```yaml
# Ensure X-Forwarded-Proto header is set
middlewares:
  - name: headers
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: "https"
```

---

## Port Forwarding Setup

For external access, configure your router:

### Required Port Forwards

| External Port | Internal IP | Internal Port | Protocol | Purpose |
|---------------|-------------|---------------|----------|---------|
| 443 | 192.168.92.42 | 443 | TCP | HTTPS (Traefik) |
| 80 (optional) | 192.168.92.42 | 80 | TCP | HTTP (redirects to HTTPS) |

**Note**: You can forward to any K3s node IP - Traefik runs on all nodes.

### DNS Configuration

In Cloudflare (or your DNS provider):

**A Records:**
```
gork.thiemo.click    → <your-public-ip>
n8n.thiemo.click     → <your-public-ip>
argocd.feto.dev      → <your-public-ip>
grafana.feto.dev     → <your-public-ip>
router.feto.dev      → <your-public-ip>
```

**Or use Cloudflare Tunnel (more secure):**
- No port forwarding needed
- TLS termination at Cloudflare
- DDoS protection included

---

## Performance Optimization

### Connection Pooling

Traefik maintains connection pools to backends:

```yaml
# Can be tuned if needed
serversTransport:
  maxIdleConnsPerHost: 200
```

### Compression

Enable gzip compression:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: compress
spec:
  compress: {}
```

Apply to routes:
```yaml
routes:
  - match: Host(`gork.thiemo.click`)
    middlewares:
      - name: compress
```

---

## Security Best Practices

### 1. Always Use HTTPS

✅ All IngressRoutes use `websecure` entrypoint
✅ HTTP routes redirect to HTTPS
✅ TLS certificates auto-renewed

### 2. Security Headers

Add security headers to all public services:

```yaml
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: "1; mode=block"
Strict-Transport-Security: "max-age=31536000"
```

### 3. Disable Traefik Dashboard

❌ Dashboard exposes cluster information
✅ Currently disabled in production

### 4. IP Whitelisting (Optional)

For internal services:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: internal-only
spec:
  ipWhiteList:
    sourceRange:
      - "192.168.0.0/16"  # LAN only
```

---

## Advanced Routing

### Path-Based Routing

```yaml
routes:
  - match: Host(`example.com`) && PathPrefix(`/api`)
    kind: Rule
    services:
      - name: backend-api
        port: 8080
  - match: Host(`example.com`) && PathPrefix(`/`)
    kind: Rule
    services:
      - name: frontend
        port: 3000
```

### Header-Based Routing

```yaml
routes:
  - match: Host(`example.com`) && Headers(`X-Version`, `v2`)
    kind: Rule
    services:
      - name: backend-v2
        port: 8080
```

### Weighted Load Balancing (Canary Deployments)

```yaml
routes:
  - match: Host(`example.com`)
    kind: Rule
    services:
      - name: backend-v1
        port: 8080
        weight: 90  # 90% traffic
      - name: backend-v2
        port: 8080
        weight: 10  # 10% traffic
```

---

## Monitoring Metrics

Traefik exposes Prometheus metrics (scraped by your monitoring stack):

**Available metrics:**
- `traefik_entrypoint_requests_total` - Total requests per entrypoint
- `traefik_service_requests_total` - Requests per service
- `traefik_entrypoint_request_duration_seconds` - Request latency
- `traefik_backend_server_up` - Backend health status

**View in Grafana:**
- Use Traefik dashboard (ID: 4475)
- Custom dashboards in your Grafana instance

---

## References

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Traefik Kubernetes CRD](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/)
- [K3s LoadBalancer](https://docs.k3s.io/networking/networking-services#service-load-balancer)

---

## Changelog

- **2025-10-26**: Documented existing Traefik ingress configuration
  - DaemonSet deployment across all nodes
  - IngressRoute examples for all services
  - Middleware configurations
  - External service proxy pattern
