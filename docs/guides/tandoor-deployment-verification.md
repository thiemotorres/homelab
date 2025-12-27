# Tandoor Recipes Deployment Verification

This guide provides verification steps to confirm the Tandoor Recipes deployment is working correctly.

## Prerequisites

- kubectl configured and connected to your cluster
- ArgoCD CLI installed (optional)
- Access to DNS configuration

## 1. Verify Pod Status

Check that the Tandoor pod is running:

```bash
kubectl get pods -n tandoor
```

Expected output:
```
NAME                       READY   STATUS    RESTARTS   AGE
tandoor-xxxxxxxxxx-xxxxx   1/1     Running   0          Xm
```

The pod should show:
- `READY: 1/1`
- `STATUS: Running`
- `RESTARTS: 0` (or low number)

## 2. Verify Database

Check that the PostgreSQL database was created:

```bash
kubectl exec -n infrastructure postgres-0 -- psql -U postgres -l | grep tandoor
```

Expected output:
```
 tandoor   | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           |
```

## 3. Verify Certificate

Check that the TLS certificate is ready:

```bash
kubectl get certificate -n tandoor
```

Expected output:
```
NAME           READY   SECRET        AGE
tandoor-cert   True    tandoor-tls   Xm
```

The certificate should show `READY: True`.

To check certificate details:

```bash
kubectl describe certificate tandoor-cert -n tandoor
```

## 4. Verify Ingress Route

Check that the Traefik IngressRoute is configured:

```bash
kubectl get ingressroute -n tandoor
```

Expected output:
```
NAME      AGE
tandoor   Xm
```

To view the IngressRoute configuration:

```bash
kubectl describe ingressroute tandoor -n tandoor
```

## 5. Verify Storage

Check that the PersistentVolumeClaim is bound:

```bash
kubectl get pvc -n tandoor
```

Expected output:
```
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
hetzner-tandoor-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   10Gi       RWO            hetzner-cifs   Xm
```

The PVC should show `STATUS: Bound`.

## 6. Verify Service

Check that the Kubernetes service is available:

```bash
kubectl get service -n tandoor
```

Expected output:
```
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
tandoor-service   ClusterIP   10.43.xxx.xxx   <none>        8080/TCP   Xm
```

## 7. Check Application Logs

View the application logs to ensure it started successfully:

```bash
kubectl logs -n tandoor deployment/tandoor --tail=50
```

You should see Gunicorn startup messages like:
```
[YYYY-MM-DD HH:MM:SS +0100] [1] [INFO] Starting gunicorn X.X.X
[YYYY-MM-DD HH:MM:SS +0100] [1] [INFO] Listening at: http://[::]:8080 (1)
[YYYY-MM-DD HH:MM:SS +0100] [1] [INFO] Using worker: gthread
[YYYY-MM-DD HH:MM:SS +0100] [XX] [INFO] Booting worker with pid: XX
```

## 8. Test HTTPS Access

Test the HTTPS endpoint:

```bash
curl -I https://rezepte.thiemo.click
```

Expected output should include:
```
HTTP/2 302
location: /setup/
```

The 302 redirect to `/setup/` indicates Tandoor is running correctly and needs initial setup.

## 9. Access Web Interface

Open your browser and navigate to:

```
https://rezepte.thiemo.click
```

You should be redirected to the setup page where you can:
1. Create the first admin user
2. Configure the default space
3. Set up initial preferences

## 10. Verify ArgoCD Sync Status (if using ArgoCD)

Check that the ArgoCD application is synced:

```bash
kubectl get application tandoor -n argocd
```

Expected output:
```
NAME      SYNC STATUS   HEALTH STATUS
tandoor   Synced        Healthy
```

For detailed sync status:

```bash
kubectl describe application tandoor -n argocd
```

## Troubleshooting

### Pod Not Running

If the pod is not running:

1. Check pod events:
   ```bash
   kubectl describe pod -n tandoor -l app=tandoor
   ```

2. Check pod logs:
   ```bash
   kubectl logs -n tandoor deployment/tandoor
   ```

### Certificate Not Ready

If the certificate is not ready:

1. Check certificate events:
   ```bash
   kubectl describe certificate tandoor-cert -n tandoor
   ```

2. Check cert-manager logs:
   ```bash
   kubectl logs -n cert-manager deployment/cert-manager
   ```

### HTTPS Not Working

If HTTPS access fails:

1. Verify DNS is pointing to your cluster
2. Check Traefik logs:
   ```bash
   kubectl logs -n traefik deployment/traefik
   ```

3. Verify the IngressRoute configuration:
   ```bash
   kubectl get ingressroute tandoor -n tandoor -o yaml
   ```

### Database Connection Issues

If the pod logs show database connection errors:

1. Verify the PostgreSQL service is running:
   ```bash
   kubectl get pods -n infrastructure | grep postgres
   ```

2. Check the sealed secrets are properly deployed:
   ```bash
   kubectl get secrets -n tandoor
   ```

3. Verify database credentials:
   ```bash
   kubectl get secret postgres-credentials -n tandoor -o yaml
   ```

## Post-Deployment Steps

After verification:

1. Complete the initial setup via the web interface
2. Create your first recipe to test functionality
3. Configure backup procedures for the database and media files
4. Set up monitoring and alerting (if not already configured)

## Related Documentation

- [Tandoor Recipes Official Documentation](https://docs.tandoor.dev/)
- [Integration Design](../designs/tandoor-recipes-integration-design.md)
- Traefik IngressRoute configuration
- cert-manager certificate management
