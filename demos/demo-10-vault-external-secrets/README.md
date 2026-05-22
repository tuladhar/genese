# Demo 10: HashiCorp Vault + External Secrets Operator

> **Duration:** ~12 minutes  
> **Section:** 4.3 — External Secrets Management  
> **Tools:** minikube, kubectl, Helm, HashiCorp Vault, External Secrets Operator

---

## Learning Objectives

- Deploy HashiCorp Vault in Kubernetes (dev mode for the demo)
- Store secrets in Vault with path-based access control
- Install the External Secrets Operator (ESO)
- Sync Vault secrets to Kubernetes Secrets automatically via ESO
- Understand the production pattern for Azure Key Vault integration

---

## Prerequisites

- Default minikube cluster running: `minikube status`
- Helm installed: `helm version`
- If needed: `minikube start --cpus=2 --memory=4096 --driver=docker`

---

## Setup

```bash
mkdir -p ~/devsecops-demo/demo-10 && cd ~/devsecops-demo/demo-10
```

---

## Part 1: Deploy HashiCorp Vault

We run Vault in **dev mode** — unsecured single-node for demo purposes. Production Vault uses HA, auto-unseal, and persistent storage.

### Step 1 — Install Vault via Helm

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set "server.dev.enabled=true" \
  --set "server.dev.devRootToken=root" \
  --set "injector.enabled=false"

# Wait for Vault to be ready
kubectl wait --for=condition=Ready pod/vault-0 -n vault --timeout=120s
echo "Vault is running."
```

### Step 2 — Verify Vault status

```bash
kubectl exec -n vault vault-0 -- vault status
```

**Expected output:**
```
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Version         1.15.x
...
```

---

## Part 2: Store Secrets in Vault

### Step 3 — Configure the Vault CLI

```bash
# Set up access to Vault (dev mode root token)
export VAULT_TOKEN="root"
export VAULT_ADDR="http://127.0.0.1:8200"

# Port-forward Vault (run in background)
kubectl port-forward -n vault vault-0 8200:8200 &
PF_PID=$!
sleep 3
echo "Vault available at http://127.0.0.1:8200"
```

### Step 4 — Enable the KV secrets engine and store secrets

```bash
# Enable KV v2 secrets engine at path "secret/"
kubectl exec -n vault vault-0 -- vault secrets enable -path=secret kv-v2 2>/dev/null || \
  echo "KV engine already enabled (dev mode default)"

# Store a database credential
kubectl exec -n vault vault-0 -- \
  vault kv put secret/myapp/database \
    username="appuser" \
    password="Vault$ecure#2025" \
    host="postgres.internal.dangote.com" \
    port="5432"

# Store an API key
kubectl exec -n vault vault-0 -- \
  vault kv put secret/myapp/api-keys \
    payment-gateway="pg_live_abc123xyz789" \
    sms-provider="sms_key_def456uvw"

echo "Secrets stored in Vault."
```

### Step 5 — Read the secrets back to confirm

```bash
kubectl exec -n vault vault-0 -- vault kv get secret/myapp/database
```

**Expected output:**
```
====== Secret Path ======
secret/data/myapp/database

======= Metadata =======
created_time    2025-01-01T10:00:00Z
version         1

====== Data ======
Key         Value
---         -----
host        postgres.internal.dangote.com
password    Vault$ecure#2025
port        5432
username    appuser
```

---

## Part 3: Store the Vault Token as a Kubernetes Secret

ESO needs a credential to authenticate to Vault. For this demo we use Vault's dev root token, stored as a Kubernetes Secret that ESO reads.

> **Production note:** In production, use Vault's Kubernetes auth method with short-lived tokens and minimal policies. The token approach here is intentionally simple for a local demo.

### Step 6 — Create the app namespace and store the Vault token

```bash
kubectl create namespace app-demo

# Store the Vault dev root token so ESO can authenticate
kubectl create secret generic vault-token \
  --from-literal=token=root \
  -n app-demo

echo "Vault token stored."
```

---

## Part 4: Install External Secrets Operator

The External Secrets Operator (ESO) watches for `ExternalSecret` resources and syncs secrets from an external store (Vault, AWS, Azure, GCP) into Kubernetes Secrets.

### Step 7 — Install ESO via Helm

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --wait

echo "External Secrets Operator installed."
kubectl get pods -n external-secrets
```

---

## Part 5: Sync Vault Secrets to Kubernetes

### Step 8 — Create a SecretStore (connection to Vault)

```bash
cat > secret-store.yaml << 'EOF'
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: app-demo
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        tokenSecretRef:
          name: vault-token      # K8s Secret holding the Vault token
          key: token
EOF

kubectl apply -f secret-store.yaml
sleep 5

# Check the store is ready
kubectl get secretstore vault-backend -n app-demo
```

**Expected output:**
```
NAME            AGE   STATUS   READY
vault-backend   5s    Valid    True
```

### Step 9 — Create an ExternalSecret to sync credentials

```bash
cat > external-secret.yaml << 'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: app-demo
spec:
  # How often to refresh from Vault
  refreshInterval: 1m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-credentials          # Name of the K8s Secret that will be created
    creationPolicy: Owner
  data:
  - secretKey: DB_USERNAME        # Key in the K8s Secret
    remoteRef:
      key: myapp/database         # Path in Vault (relative to the engine path)
      property: username          # Property within the secret
  - secretKey: DB_PASSWORD
    remoteRef:
      key: myapp/database
      property: password
  - secretKey: DB_HOST
    remoteRef:
      key: myapp/database
      property: host
EOF

kubectl apply -f external-secret.yaml
sleep 10
```

### Step 10 — Verify the Kubernetes Secret was created

```bash
kubectl get externalsecret database-credentials -n app-demo
```

**Expected output:**
```
NAME                   STORE           REFRESH INTERVAL   STATUS         READY
database-credentials   vault-backend   1m                 SecretSynced   True
```

```bash
# The K8s Secret was automatically created by ESO
kubectl get secret db-credentials -n app-demo
kubectl get secret db-credentials -n app-demo -o jsonpath='{.data.DB_HOST}' | base64 -d
echo ""
```

**Expected output:**
```
postgres.internal.dangote.com
```

The secret is now available as a standard Kubernetes Secret — applications consume it normally without knowing Vault is involved.

### Step 11 — Simulate secret rotation in Vault

```bash
# Update the password in Vault
kubectl exec -n vault vault-0 -- \
  vault kv put secret/myapp/database \
    username="appuser" \
    password="NewRotated$ecure#2025" \
    host="postgres.internal.dangote.com" \
    port="5432"

echo "Waiting for ESO to sync (refresh interval: 1m)..."
sleep 65

# The K8s Secret should now have the new password
kubectl get secret db-credentials -n app-demo \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
echo ""
```

The password in the Kubernetes Secret rotated automatically — no pod restart needed, no manual intervention.

---

## Azure Key Vault: Architecture Reference

For production environments, the same ESO pattern works with Azure Key Vault:

```yaml
# SecretStore for Azure Key Vault (requires Azure credentials)
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: azure-keyvault
  namespace: app-demo
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity    # Use Azure AD Workload Identity (recommended)
      vaultUrl: "https://dangote-prod.vault.azure.net"
      tenantId: "<azure-tenant-id>"
```

The `ExternalSecret` resource is identical regardless of the backend (Vault, Azure, AWS).

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  External Secrets Operator                       │   │
│  │  ┌──────────────┐     ┌──────────────────────┐  │   │
│  │  │ ExternalSecret│────►│  SecretStore         │  │   │
│  │  │  (declares    │     │  (Vault / Azure /    │  │   │
│  │  │   what to sync│     │   AWS credentials)   │  │   │
│  │  └──────────────┘     └──────────┬───────────┘  │   │
│  │                                  │               │   │
│  │  ┌─────────────────┐             │               │   │
│  │  │  K8s Secret     │◄────────────┘               │   │
│  │  │  (auto-created  │      syncs every 1 min      │   │
│  │  │   and rotated)  │                             │   │
│  └──┴─────────────────┴─────────────────────────────┘   │
│              │                                           │
│              ▼                                           │
│  ┌──────────────────────┐      ┌──────────────────┐     │
│  │  Application Pod     │      │  HashiCorp Vault  │     │
│  │  (consumes as normal │      │  (source of truth │     │
│  │   K8s Secret)        │      │   for all secrets)│     │
│  └──────────────────────┘      └──────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

---

## Cleanup

```bash
kubectl delete namespace app-demo external-secrets vault
helm uninstall vault -n vault 2>/dev/null || true
helm uninstall external-secrets -n external-secrets 2>/dev/null || true
rm -rf ~/devsecops-demo/demo-10
```

---

## Key Takeaways

1. **External secret managers** (Vault, Azure Key Vault) store secrets outside Kubernetes — no secrets in etcd.
2. The **External Secrets Operator** bridges the gap: your apps use normal K8s Secrets, but the source is Vault.
3. **Secret rotation** in Vault automatically propagates to Kubernetes within the refresh interval — no manual updates.
4. **Token authentication** in this demo uses a Vault token stored as a K8s Secret — production should use Kubernetes auth or cloud IAM for short-lived credentials.
5. This pattern works identically with Azure Key Vault, AWS Secrets Manager, and GCP Secret Manager — just change the SecretStore provider.
