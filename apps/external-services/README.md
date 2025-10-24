# External Services Configuration

This directory contains configurations for exposing external services (running outside the Kubernetes cluster) through your cluster's ingress with TLS certificates.

## How It Works

1. **External Service**: A Kubernetes Service with manual Endpoints that point to external IP addresses
2. **Certificate**: Let's Encrypt certificate for the subdomain (using Cloudflare DNS-01 challenge)
3. **IngressRoute**: Traefik configuration that routes traffic from the subdomain to the external service

## Current Services

### Router (router.feto.dev)
- **Target**: 192.168.92.1:440 (HTTPS)
- **Purpose**: Access your router's web interface through a clean domain with proper TLS
- **URL**: https://router.feto.dev

## Adding New External Services

1. **Copy the template**: Use `TEMPLATE.yaml` as a starting point
2. **Update the configuration**:
   - Change service name and labels
   - Update the external IP address and port
   - Set the desired subdomain (*.feto.dev)
   - Adjust the backend scheme (http/https)
3. **Add to kustomization.yaml**
4. **Commit and let ArgoCD sync**

### Example: Adding a NAS

```yaml
# nas-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nas-service
  namespace: external-services
spec:
  type: ClusterIP
  ports:
  - port: 5000
    targetPort: 5000
    protocol: TCP
    name: http
---
apiVersion: v1
kind: Endpoints
metadata:
  name: nas-service
  namespace: external-services
subsets:
- addresses:
  - ip: 192.168.92.100  # Your NAS IP
  ports:
  - port: 5000
    name: http
    protocol: TCP
---
# Certificate and IngressRoute similar to router example
```

## Benefits

✅ **Clean URLs**: Use friendly subdomains instead of IP:PORT  
✅ **TLS Encryption**: Automatic Let's Encrypt certificates  
✅ **Single Entry Point**: All services accessible through your domain  
✅ **Access Control**: Can add authentication middlewares if needed  
✅ **Monitoring**: All traffic goes through Traefik for observability  

## DNS Configuration

Make sure your DNS (Cloudflare) has A records pointing to your cluster's external IP:
- `*.feto.dev` → Your cluster's external IP (where Traefik runs)

## Security Notes

- External services are exposed with their original security
- Consider adding Traefik middleware for:
  - IP whitelisting (`ipWhiteList`)
  - Basic authentication (`basicAuth`)
  - OAuth/OIDC forward auth
- Monitor access logs through Traefik

## Troubleshooting

### Certificate Issues
```bash
kubectl get certificate -n external-services
kubectl describe certificate router-cert -n external-services
```

### Traefik Issues
```bash
kubectl logs -n kube-system deployment/traefik
```

### Service Connectivity
```bash
# Test if external service is reachable from cluster
kubectl run debug --image=nicolaka/netshoot -it --rm -- curl -k https://192.168.92.1:440
```