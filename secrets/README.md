# Secrets Management

Dieses Verzeichnis enthält Templates und Anleitungen für Secrets.

## Cloudflare API Tokens

Wir haben **zwei Domains** mit separaten API Tokens:
- **feto.dev** - Local services (internes Netzwerk)
- **thiemo.click** - Internet-facing services (öffentlich)

### 1. Cloudflare API Tokens erstellen

**Für jede Domain** einen separaten Token erstellen:

1. Gehe zu https://dash.cloudflare.com/profile/api-tokens
2. Klicke auf "Create Token"
3. Wähle "Edit zone DNS" Template (oder custom)
4. Konfiguration:
   - **Permissions:**
     - Zone > DNS > Edit
     - Zone > Zone > Read
   - **Zone Resources:**
     - Include > Specific zone > `feto.dev` (oder `thiemo.click`)
5. Kopiere den Token (wird nur einmal angezeigt!)
6. Wiederhole für die andere Domain

**Warum separate Tokens?** Bessere Sicherheit - wenn ein Token kompromittiert wird, betrifft es nur eine Domain.

### 2. Secrets erstellen

**Option A: Via YAML Dateien** (NICHT committen!)

`cloudflare-api-token-feto.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-feto
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "token-für-feto-dev-hier"
```

`cloudflare-api-token-thiemo.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-thiemo
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "token-für-thiemo-click-hier"
```

**Option B: Direkt via kubectl** (empfohlen für Start)

```bash
# Namespace erstellen
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

### 3. Secrets überprüfen

```bash
# Beide Secrets existieren?
kubectl get secrets -n cert-manager | grep cloudflare

# Sollte zeigen:
# cloudflare-api-token-feto    Opaque   1      Xs
# cloudflare-api-token-thiemo  Opaque   1      Xs

# Inhalt anzeigen (zum Debuggen)
kubectl get secret cloudflare-api-token-feto -n cert-manager -o yaml
kubectl get secret cloudflare-api-token-thiemo -n cert-manager -o yaml
```

## 🔐 Secrets verschlüsselt in Git speichern (EMPFOHLEN!)

**Manuelle Secrets sind für Quick-Start OK, aber für Production solltest du Sealed Secrets verwenden!**

### ✅ Sealed Secrets (Empfohlen - bereits deployed!)

Der Sealed Secrets Controller wird **automatisch via ArgoCD** deployed (`infrastructure/sealed-secrets/`).

**Workflow:**

1. **kubeseal CLI installieren:**
   ```bash
   brew install kubeseal  # macOS
   ```

2. **Public Key vom Cluster holen:**
   ```bash
   kubeseal --fetch-cert --controller-namespace=kube-system > pub-cert.pem
   ```

3. **Secrets verschlüsseln:**
   ```bash
   # Erstelle zuerst unverschlüsselte YAML (z.B. in /tmp/)
   # Dann verschlüsseln:
   kubeseal --format=yaml --cert=pub-cert.pem \
     < /tmp/cloudflare-feto.yaml \
     > infrastructure/cert-manager/sealed-secrets/cloudflare-api-token-feto-sealed.yaml

   kubeseal --format=yaml --cert=pub-cert.pem \
     < /tmp/cloudflare-thiemo.yaml \
     > infrastructure/cert-manager/sealed-secrets/cloudflare-api-token-thiemo-sealed.yaml
   ```

4. **In Git committen:**
   ```bash
   git add infrastructure/cert-manager/sealed-secrets/*.yaml
   git commit -m "Add sealed Cloudflare API tokens"
   git push
   ```

5. **ArgoCD synced automatisch** und der Controller entschlüsselt die Secrets!

**📖 Detaillierte Anleitung:** Siehe `SEALED-SECRETS-WORKFLOW.md`

### Option B: External Secrets Operator

Secrets aus externen Vaults (Vault, AWS Secrets Manager, etc.) syncen.

**Installation:**

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace
```

**Verwendung mit Vault/1Password/etc.** (siehe Docs)

### Option C: SOPS mit age

```bash
# SOPS und age installieren
brew install sops age

# Age Key generieren
age-keygen -o age.key
# Zeigt: public key: age1xxxxxxxxxx

# Secret verschlüsseln
sops --encrypt \
  --age age1xxxxxxxxxx \
  cloudflare-api-token.yaml > cloudflare-api-token.enc.yaml

# In Git committen
git add cloudflare-api-token.enc.yaml

# Entschlüsseln (lokal)
sops --decrypt cloudflare-api-token.enc.yaml | kubectl apply -f -
```

**Wichtig:** Den `age.key` Private Key NICHT ins Git! Sicher aufbewahren.

## ArgoCD Integration

Für SOPS mit ArgoCD brauchst du das KSOPS Plugin oder Helm Secrets.

Für Sealed Secrets funktioniert ArgoCD out-of-the-box - einfach die SealedSecret-Manifeste ins Git pushen!

## Nächste Schritte

1. ✅ Cloudflare API Token erstellen
2. ✅ Secret ins Cluster deployen
3. ✅ ClusterIssuer testen mit Staging
4. ✅ Auf Production umstellen
5. 🔐 (Optional) Sealed Secrets einrichten
