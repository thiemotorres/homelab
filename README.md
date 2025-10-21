# Homelab K3s + ArgoCD Setup

GitOps Repository für K3s Cluster mit ArgoCD, Traefik Ingress und cert-manager mit **Cloudflare** für Let's Encrypt DNS-01 Challenges.

## 🌐 Domains

- **feto.dev** - Local services (internes Netzwerk, nicht öffentlich erreichbar)
- **thiemo.click** - Internet-facing services (öffentlich erreichbar)

Beide Domains verwenden Cloudflare DNS für automatische Let's Encrypt Zertifikate.

## 🏗️ Struktur

```
.
├── bootstrap/
│   └── argocd/              # ArgoCD Installation via Helm
├── argocd-apps/             # ArgoCD Applications (App of Apps Pattern)
│   ├── root-app.yaml        # Root Application
│   ├── infrastructure.yaml  # Infrastructure Apps
│   └── apps.yaml            # User Applications
├── infrastructure/          # Infrastruktur-Komponenten
│   ├── namespaces/          # Namespace Definitionen
│   ├── sealed-secrets/      # Sealed Secrets Controller
│   ├── traefik/             # Traefik Ingress Controller
│   └── cert-manager/        # cert-manager + ClusterIssuers (Cloudflare)
├── apps/                    # Applikationen
│   ├── whoami/              # Beispiel-App (feto.dev - local)
│   └── hello-world/         # Beispiel-App (thiemo.click - public)
└── secrets/                 # Secrets Management
    ├── README.md
    └── SEALED-SECRETS-WORKFLOW.md
```

## 🚀 Schnellstart

### 0. Cloudflare DNS Setup

**Beide Domains zu Cloudflare migrieren:**

1. **Domains zu Cloudflare transferieren** ODER **Nameserver ändern**:
   - Bei Namecheap (für beide Domains): Domain Management → Nameservers → Custom DNS
   - Setze Cloudflare Nameservers (z.B. `ns1.cloudflare.com`, `ns2.cloudflare.com`)
   - Für **feto.dev** UND **thiemo.click**
   - Warte bis DNS propagiert ist (kann bis 24h dauern)

2. **Cloudflare API Tokens erstellen** (einen Token pro Domain):

   **Token für feto.dev:**
   - Gehe zu https://dash.cloudflare.com/profile/api-tokens
   - "Create Token" → "Edit zone DNS" Template
   - Permissions: `Zone:DNS:Edit` + `Zone:Zone:Read`
   - Zone Resources: `feto.dev`
   - Token kopieren und sicher aufbewahren!

   **Token für thiemo.click:**
   - Gleicher Prozess, aber Zone: `thiemo.click`
   - Wieder Token kopieren

   **Tipp:** Separate Tokens erhöhen die Sicherheit - falls einer kompromittiert wird, ist nur eine Domain betroffen.

### 1. ArgoCD installieren

```bash
# Namespace erstellen
kubectl create namespace argocd

# ArgoCD via Helm installieren
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --values bootstrap/argocd/values.yaml
```

Alternativ mit Kustomize:
```bash
kubectl apply -k bootstrap/argocd/
```

### 2. ArgoCD UI zugreifen

Initial-Passwort abrufen:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Port-Forward (temporär):
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Öffne: https://localhost:8080
- Username: `admin`
- Password: (siehe oben)

### 3. Cloudflare Secrets erstellen

```bash
# Namespace für cert-manager (wird später automatisch erstellt, aber wir brauchen es jetzt schon)
kubectl create namespace cert-manager

# Secret für feto.dev
kubectl create secret generic cloudflare-api-token-feto \
  --namespace cert-manager \
  --from-literal=api-token='TOKEN_FETO_DEV'

# Secret für thiemo.click
kubectl create secret generic cloudflare-api-token-thiemo \
  --namespace cert-manager \
  --from-literal=api-token='TOKEN_THIEMO_CLICK'
```

Siehe `secrets/README.md` für Details und verschlüsselte Variante (Sealed Secrets).

### 4. Root App deployen

```bash
kubectl apply -f argocd-apps/root-app.yaml
```

ArgoCD wird jetzt automatisch:
1. Infrastructure deployen (Namespaces, Traefik, cert-manager)
2. Apps deployen (whoami auf feto.dev, hello-world auf thiemo.click)

### 5. Sync im ArgoCD UI beobachten

Öffne ArgoCD UI und schaue zu wie die Applications synced werden:
- `root` → deployed `infrastructure` und `apps`
- `infrastructure` → deployed Traefik + cert-manager
- `apps` → deployed whoami (feto.dev) + hello-world (thiemo.click)

### 6. Zertifikat überprüfen

```bash
# Certificate Status
kubectl get certificate -n whoami

# Sollte nach ~2 Minuten "Ready" sein
kubectl describe certificate whoami-cert -n whoami

# Bei Problemen: cert-manager Logs
kubectl logs -n cert-manager deployment/cert-manager -f
```

### 7. Testen

```bash
# DNS auflösen
dig whoami.feto.dev
dig hello.thiemo.click

# HTTPS testen (nach DNS Propagation)
curl -v https://whoami.feto.dev         # Local service
curl -v https://hello.thiemo.click      # Public service
```

## 📋 Wichtige Kommandos

### ArgoCD CLI

```bash
# ArgoCD CLI installieren (macOS)
brew install argocd

# Login
argocd login localhost:8080

# Apps anzeigen
argocd app list

# App sync
argocd app sync infrastructure
argocd app sync apps

# App logs
argocd app logs whoami
```

### Debugging

```bash
# Certificate Status
kubectl get certificate -A
kubectl describe certificate whoami-cert -n whoami

# Certificate Request (bei Problemen)
kubectl get certificaterequest -n whoami
kubectl describe certificaterequest -n whoami

# Challenge (DNS-01)
kubectl get challenge -n whoami
kubectl describe challenge -n whoami

# cert-manager Logs
kubectl logs -n cert-manager deployment/cert-manager -f

# Traefik Logs
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik -f

# ArgoCD Application Status
kubectl get applications -n argocd
kubectl describe application infrastructure -n argocd
```

## 🔧 Konfiguration

### Traefik

K3s installiert Traefik standardmäßig. Unser Setup:
- Deployed via Helm in `infrastructure/traefik/`
- Verwendet `DaemonSet` statt Deployment
- LoadBalancer Service (ändere auf NodePort wenn kein MetalLB vorhanden)

Falls du K3s **ohne** Traefik installieren willst:
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
```

### cert-manager mit Cloudflare

**Vorteile:**
- ✅ Nativ unterstützt (kein Webhook nötig!)
- ✅ Sehr zuverlässig
- ✅ Wildcard-Zertifikate möglich
- ✅ Schnelle DNS-Propagation

**Staging vs Production:**
- Immer zuerst mit `letsencrypt-staging` testen!
- Erst nach erfolgreichem Test auf `letsencrypt-prod` umstellen
- Production hat Rate Limits (50 Certs/Woche pro Domain)

**ClusterIssuer anpassen:**

In `infrastructure/cert-manager/clusterissuer-*.yaml` deine E-Mail eintragen:
```yaml
email: info@thiemo-torres.dem  # ÄNDERN!
```

### ArgoCD Sync Policy

Alle Apps haben `automated` sync aktiviert:
```yaml
syncPolicy:
  automated:
    prune: true      # Löscht Ressourcen die nicht mehr im Git sind
    selfHeal: true   # Korrigiert manuelle Änderungen automatisch
```

### Sealed Secrets

**Automatisch deployed!** Der Sealed Secrets Controller wird via ArgoCD in `infrastructure/sealed-secrets/` deployed.

**Workflow:**
1. Cloudflare Tokens manuell erstellen (für Quick-Start)
2. `kubeseal` CLI installieren: `brew install kubeseal`
3. Secrets verschlüsseln (siehe `secrets/SEALED-SECRETS-WORKFLOW.md`)
4. Verschlüsselte Secrets ins Git committen
5. ArgoCD synced → Controller entschlüsselt → cert-manager nutzt Secrets

**Vorteil:** Secrets sicher im Git speichern, vollständig GitOps-konform!

## 🔐 Secrets Management

### Quick-Start (Manuell)

Für den ersten Test: Secrets manuell erstellen (siehe Schritt 3 im Schnellstart oben).

### Production (Sealed Secrets - EMPFOHLEN!)

**Bereits konfiguriert!** Sealed Secrets Controller wird automatisch deployed.

**Nächste Schritte:**
1. Siehe `secrets/SEALED-SECRETS-WORKFLOW.md` für detaillierte Anleitung
2. Cloudflare Tokens verschlüsseln
3. Ins Git committen
4. Fertig!

**Alternativen:**
- **External Secrets Operator** - Anbindung an Vault, 1Password, AWS Secrets Manager
- **SOPS** - Verschlüsseln mit age/GPG

Details siehe `secrets/README.md`.

## 📦 Neue Apps hinzufügen

### Variante 1: Zu apps/ hinzufügen

1. Erstelle Ordner `apps/my-app/`
2. Erstelle Manifests + `kustomization.yaml`
3. Füge zu `apps/kustomization.yaml` hinzu:
   ```yaml
   resources:
     - whoami
     - my-app  # NEU
   ```
4. Git commit & push → ArgoCD synced automatisch!

### Variante 2: Eigene ArgoCD Application

Erstelle `argocd-apps/my-app.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/thiemotorres/homelab.git
    targetRevision: main
    path: apps/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## 🌐 Wildcard-Zertifikate

Für `*.feto.dev` Wildcard-Cert:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-feto-dev
  namespace: default
spec:
  secretName: wildcard-feto-dev-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - "*.feto.dev"
    - "feto.dev"  # Auch Root-Domain
```

Dann in IngressRoutes:
```yaml
tls:
  secretName: wildcard-feto-dev-tls
```

## 🐛 Troubleshooting

### Certificate bleibt pending

```bash
# Challenge prüfen
kubectl get challenge -A
kubectl describe challenge -n whoami

# cert-manager Logs
kubectl logs -n cert-manager deployment/cert-manager -f
```

Häufige Probleme:
- **Secret fehlt:** Cloudflare API Token nicht vorhanden
- **Token falsch:** Permissions prüfen (Zone:DNS:Edit + Zone:Zone:Read)
- **DNS nicht propagiert:** Nameserver bei Namecheap noch nicht auf Cloudflare
- **Rate Limit:** Staging Issuer verwenden zum Testen!

### ArgoCD App "OutOfSync"

```bash
# Status checken
argocd app get infrastructure

# Diff anzeigen
argocd app diff infrastructure

# Force sync
argocd app sync infrastructure --force
```

### Traefik erreicht Service nicht

```bash
# IngressRoute prüfen
kubectl get ingressroute -A

# Traefik Logs
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik -f

# Service und Pods prüfen
kubectl get svc,pods -n whoami
```

### DNS funktioniert nicht

```bash
# Cloudflare Nameserver prüfen
dig NS feto.dev

# A-Record prüfen
dig whoami.feto.dev

# Bei Cloudflare: Proxy Status prüfen
# Orange Cloud = Proxied (kann Probleme mit cert-manager machen)
# Gray Cloud = DNS Only (empfohlen für Let's Encrypt)
```

## 🎯 Nächste Schritte

- [x] K3s Cluster
- [x] ArgoCD
- [x] Traefik Ingress
- [x] cert-manager mit Cloudflare
- [x] Sealed Secrets Controller
- [x] Beispiel-Apps (whoami, hello-world)
- [ ] Cloudflare Tokens mit Sealed Secrets verschlüsseln
- [ ] Monitoring (Prometheus/Grafana)
- [ ] Logging (Loki)
- [ ] Backup (Velero)
- [ ] ArgoCD Notifications (Slack/Discord)

## 📚 Links

- [ArgoCD Docs](https://argo-cd.readthedocs.io/)
- [cert-manager Docs](https://cert-manager.io/docs/)
- [Traefik Docs](https://doc.traefik.io/traefik/)
- [K3s Docs](https://docs.k3s.io/)
- [Cloudflare Docs](https://developers.cloudflare.com/)
