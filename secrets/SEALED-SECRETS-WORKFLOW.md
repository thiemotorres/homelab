# Sealed Secrets Workflow

## Warum Sealed Secrets?

✅ **Sicher in Git** - Verschlüsselt, nur dein Cluster kann entschlüsseln
✅ **GitOps-Ready** - ArgoCD synced automatisch
✅ **Einfach** - Nur ein CLI Tool (`kubeseal`)
✅ **Transparent** - Wird zu normalem Secret im Cluster

## 🚀 Setup

### 1. Sealed Secrets Controller ist deployed

Der Controller wird automatisch via ArgoCD deployed (`infrastructure/sealed-secrets/`).

Prüfen:
```bash
kubectl get pods -n kube-system | grep sealed-secrets
# Sollte zeigen: sealed-secrets-controller-xxx   1/1   Running
```

### 2. kubeseal CLI installieren

**macOS:**
```bash
brew install kubeseal
```

**Linux:**
```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.0/kubeseal-0.27.0-linux-amd64.tar.gz
tar xfz kubeseal-0.27.0-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

**Windows:**
```powershell
# Via Scoop
scoop install kubeseal

# Oder Download von GitHub Releases
# https://github.com/bitnami-labs/sealed-secrets/releases
```

### 3. Public Key vom Cluster abrufen

```bash
kubeseal --fetch-cert --controller-namespace=kube-system > pub-cert.pem
```

Dieser Public Key wird zum Verschlüsseln verwendet. **Kann ins Git!**

## 🔐 Cloudflare Secrets verschlüsseln

### Schritt 1: Unverschlüsselte Secrets erstellen (lokal, NICHT committen!)

`/tmp/cloudflare-feto.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-feto
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "dein-echter-cloudflare-token-feto"
```

`/tmp/cloudflare-thiemo.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-thiemo
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "dein-echter-cloudflare-token-thiemo"
```

### Schritt 2: Secrets verschlüsseln

```bash
# Mit Public Key vom Cluster
kubeseal --format=yaml --cert=pub-cert.pem \
  < /tmp/cloudflare-feto.yaml \
  > infrastructure/cert-manager/sealed-secrets/cloudflare-api-token-feto-sealed.yaml

kubeseal --format=yaml --cert=pub-cert.pem \
  < /tmp/cloudflare-thiemo.yaml \
  > infrastructure/cert-manager/sealed-secrets/cloudflare-api-token-thiemo-sealed.yaml
```

**Oder direkt vom Cluster (wenn kubectl verbunden):**
```bash
kubeseal --format=yaml --controller-namespace=kube-system \
  < /tmp/cloudflare-feto.yaml \
  > infrastructure/cert-manager/sealed-secrets/cloudflare-api-token-feto-sealed.yaml

kubeseal --format=yaml --controller-namespace=kube-system \
  < /tmp/cloudflare-thiemo.yaml \
  > infrastructure/cert-manager/sealed-secrets/cloudflare-api-token-thiemo-sealed.yaml
```

### Schritt 3: SealedSecrets ins Git

Die verschlüsselten Secrets sehen ungefähr so aus:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: cloudflare-api-token-feto
  namespace: cert-manager
spec:
  encryptedData:
    api-token: AgBx4f5t... # Sehr langer verschlüsselter String
  template:
    metadata:
      name: cloudflare-api-token-feto
      namespace: cert-manager
    type: Opaque
```

**Diese sind sicher!** Du kannst sie committen:

```bash
git add infrastructure/cert-manager/sealed-secrets/*.yaml
git commit -m "Add sealed Cloudflare API tokens"
git push
```

### Schritt 4: In Kustomization einbinden

`infrastructure/cert-manager/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: cert-manager
resources:
  - clusterissuer-staging.yaml
  - clusterissuer-prod.yaml
  - sealed-secrets/cloudflare-api-token-feto-sealed.yaml
  - sealed-secrets/cloudflare-api-token-thiemo-sealed.yaml

helmCharts:
  - name: cert-manager
    repo: https://charts.jetstack.io
    version: v1.16.2
    releaseName: cert-manager
    namespace: cert-manager
    valuesFile: values.yaml
```

### Schritt 5: ArgoCD synced automatisch

Nach `git push` wird ArgoCD:
1. SealedSecret im Cluster erstellen
2. Sealed Secrets Controller entschlüsselt es
3. Normales Secret wird erstellt
4. cert-manager kann es nutzen!

## 🔍 Überprüfung

```bash
# SealedSecret existiert?
kubectl get sealedsecrets -n cert-manager

# Wurde entschlüsselt?
kubectl get secrets -n cert-manager | grep cloudflare

# Sollte beide Secrets zeigen:
# cloudflare-api-token-feto    Opaque   1      Xs
# cloudflare-api-token-thiemo  Opaque   1      Xs
```

## 🔄 Secret aktualisieren

1. Ändere das unverschlüsselte Secret lokal (`/tmp/cloudflare-*.yaml`)
2. Neu verschlüsseln mit `kubeseal`
3. SealedSecret im Git ersetzen
4. Commit & Push
5. ArgoCD synced automatisch

**Wichtig:** Du kannst das gleiche Secret mehrfach verschlüsseln, es entsteht aber jedes Mal ein anderer verschlüsselter String (weil ein Salt verwendet wird). Das ist normal!

## 🔐 Secret rotieren

Falls ein Token kompromittiert wurde:

1. **Neuen Token in Cloudflare erstellen**
2. **Altes Secret löschen:**
   ```bash
   kubectl delete secret cloudflare-api-token-feto -n cert-manager
   ```
3. **Neues Secret verschlüsseln** (siehe Schritt 2 oben)
4. **Git commit & push**
5. **Alten Token in Cloudflare löschen**

## 💡 Best Practices

### Originale Secrets sicher löschen

```bash
# Nach dem Verschlüsseln!
rm /tmp/cloudflare-*.yaml

# Oder nutze shred (Linux)
shred -u /tmp/cloudflare-*.yaml
```

### Public Key im Git speichern

```bash
# Für andere Team-Mitglieder
mv pub-cert.pem secrets/sealed-secrets-pub-cert.pem
git add secrets/sealed-secrets-pub-cert.pem
git commit -m "Add sealed-secrets public key"
```

Dann können andere ohne Cluster-Zugriff Secrets verschlüsseln!

### Backup des Private Keys

Der **Private Key** ist im Cluster als Secret:

```bash
kubectl get secret -n kube-system sealed-secrets-key -o yaml > sealed-secrets-private-key-BACKUP.yaml
```

**Sehr wichtig:** Diesen Key sicher aufbewahren (KeePass, 1Password, Vault)!
Ohne diesen Key kannst du die SealedSecrets nie wieder entschlüsseln.

**NIEMALS ins Git committen!**

## 🆘 Troubleshooting

### SealedSecret wird nicht entschlüsselt

```bash
# Controller Logs checken
kubectl logs -n kube-system -l app.kubernetes.io/name=sealed-secrets -f
```

Häufige Probleme:
- Namespace stimmt nicht (muss beim Verschlüsseln und im Manifest gleich sein)
- Controller nicht gestartet
- Falscher Public Key verwendet

### Secret existiert nicht nach Sync

```bash
# SealedSecret vorhanden?
kubectl get sealedsecrets -n cert-manager

# Events checken
kubectl get events -n cert-manager --sort-by='.lastTimestamp'
```

### Neu verschlüsseln nach Cluster-Neuaufbau

Falls du den Cluster neu aufsetzt:
1. Backup des Private Keys einspielen
2. Oder alle Secrets neu erstellen und verschlüsseln

## 📚 Links

- [Sealed Secrets GitHub](https://github.com/bitnami-labs/sealed-secrets)
- [Sealed Secrets Docs](https://sealed-secrets.netlify.app/)
