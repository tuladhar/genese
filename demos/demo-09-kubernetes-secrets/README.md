# Demo 09: Kubernetes Secrets

> **Duration:** ~8 minutes  
> **Section:** 4.1–4.2 — Secrets Management Fundamentals, Kubernetes Secrets  
> **Tools:** minikube, kubectl

---

## Learning Objectives

- Understand that Kubernetes Secrets are base64-encoded, not encrypted by default
- Create and consume secrets correctly (environment variables and volume mounts)
- Identify common secret exposure mistakes
- Configure Encryption at Rest for Secrets

---

## Prerequisites

- Default minikube cluster running: `minikube status`
- If needed: `minikube start --cpus=2 --memory=4096 --driver=docker`

---

## Setup

```bash
mkdir -p ~/devsecops-demo/demo-09 && cd ~/devsecops-demo/demo-09
kubectl create namespace secrets-demo
```

---

## Part 1: The Dangerous Misconceptions

### Step 1 — Demonstrate that base64 is NOT encryption

```bash
echo "=== base64 is trivially reversible ==="
echo -n "SuperSecretPassword123" | base64
echo ""
echo "Decoding it back:"
echo "U3VwZXJTZWNyZXRQYXNzd29yZDEyMw==" | base64 -d
echo ""
```

**Expected output:**
```
U3VwZXJTZWNyZXRQYXNzd29yZDEyMw==
SuperSecretPassword123
```

Anyone with `kubectl get secret` access can read your secrets in plaintext.

### Step 2 — Create a secret and demonstrate the exposure

```bash
# Create a secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecretPassword123 \
  -n secrets-demo

# Look at what's stored in etcd (via the API)
kubectl get secret db-credentials -n secrets-demo -o yaml
```

**Expected output:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: secrets-demo
data:
  password: U3VwZXJTZWNyZXRQYXNzd29yZDEyMw==   # base64 only!
  username: YWRtaW4=
type: Opaque
```

```bash
# Decode it — anyone with kubectl access can do this
kubectl get secret db-credentials -n secrets-demo \
  -o jsonpath='{.data.password}' | base64 -d
echo ""
```

---

## Part 2: Common Secret Exposure Mistakes

### Mistake 1 — Secrets hardcoded in Docker images

```bash
cat > Dockerfile.bad-secret << 'EOF'
FROM alpine:3.19
# BAD: password baked into the image layer forever
ENV DB_PASSWORD=SuperSecretPassword123
RUN echo "database_password=$DB_PASSWORD" > /app/config.ini
CMD ["cat", "/app/config.ini"]
EOF

echo "Anyone with 'docker inspect' can see ENV vars:"
echo "  docker inspect <image> | grep ENV"
echo "  docker history <image>  # shows RUN commands with secrets"
```

### Mistake 2 — Secrets in YAML committed to Git

```bash
cat > secret-in-git.yaml << 'EOF'
# BAD: Never commit this to git
apiVersion: v1
kind: Secret
metadata:
  name: bad-secret
data:
  # This is just base64. Anyone with repo access can read it.
  api-key: c3VwZXJzZWNyZXRhcGlrZXkxMjM=
EOF

echo ""
echo "If this is committed to git, it is compromised — forever."
echo "Even if deleted, it remains in git history."
echo ""
```

---

## Part 3: Correct Secret Consumption

### Step 3 — Consume secrets as environment variables

```bash
cat > pod-env-secret.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-env-secret
  namespace: secrets-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo DB_USER=$DB_USER && echo DB_PASS is set: $(test -n \"$DB_PASS\" && echo yes || echo no) && sleep 3600"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: false
  restartPolicy: Never
EOF

kubectl apply -f pod-env-secret.yaml
kubectl wait --for=condition=Ready pod/pod-env-secret -n secrets-demo --timeout=30s
kubectl logs pod/pod-env-secret -n secrets-demo
```

**Expected output:**
```
DB_USER=admin
DB_PASS is set: yes
```

### Step 4 — Consume secrets as mounted files (preferred for sensitive values)

Volume mounts are preferred over env vars because:
- Env vars can leak into child processes and crash dumps
- Files can have restricted OS permissions
- Files can be updated without restarting the pod

```bash
cat > pod-volume-secret.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-volume-secret
  namespace: secrets-demo
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000       # Sets group ownership of volume mounts to 1000
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Password file contents:' && cat /etc/db-creds/password && echo && sleep 3600"]
    volumeMounts:
    - name: db-creds
      mountPath: /etc/db-creds
      readOnly: true
    securityContext:
      allowPrivilegeEscalation: false
  volumes:
  - name: db-creds
    secret:
      secretName: db-credentials
      defaultMode: 0440   # Owner+group read; uid 1000 (in fsGroup 1000) can read
  restartPolicy: Never
EOF

kubectl apply -f pod-volume-secret.yaml
kubectl wait --for=condition=Ready pod/pod-volume-secret -n secrets-demo --timeout=30s
kubectl logs pod/pod-volume-secret -n secrets-demo
```

**Expected output:**
```
Password file contents:
SuperSecretPassword123
```

```bash
# Verify file permissions
kubectl exec -n secrets-demo pod/pod-volume-secret -- ls -la /etc/db-creds/
```

**Expected output:**
```
lrwxrwxrwx    1 root     root    15 Jan  1 password -> ..data/password
-r--------    1 1000     1000    22 Jan  1 ..data/password
```

---

## Part 4: Encryption at Rest

By default, secrets in etcd are stored as base64. Enabling encryption at rest means they are encrypted in etcd storage.

### Step 5 — Check if encryption at rest is enabled

```bash
# Check if the API server has encryption configured
minikube ssh "sudo ps aux | grep apiserver" | grep -o "encryption-provider-config[^ ]*" || \
  echo "Encryption at rest: NOT configured (minikube default)"
```

### Step 6 — Understand what encryption at rest looks like (conceptual)

```bash
cat > encryption-config-example.yaml << 'EOF'
# This file would be passed to: --encryption-provider-config=<path>
# This is for reference only — do not apply this in the demo
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    # First provider is used for encryption
    - aescbc:
        keys:
        - name: key1
          # 32-byte base64-encoded key (generated with: head -c 32 /dev/urandom | base64)
          secret: <base64-encoded-32-byte-key>
    # Allows reading secrets not yet encrypted
    - identity: {}
EOF

echo "With this config, secrets are AES-CBC encrypted in etcd."
echo "Even if an attacker dumps etcd, secrets are unreadable without the key."
```

---

## Secret Comparison: Bad vs Good

| Pattern | Risk | Better Approach |
|---------|------|-----------------|
| Secret in Dockerfile ENV | Baked into image layers forever | Inject at runtime via K8s Secret |
| Secret in YAML committed to git | Permanent exposure | Use Sealed Secrets or external vault |
| Secret as env var in container | Leaks to child processes, logs | Use volume mount with `defaultMode: 0400` |
| Secret in log output | Log aggregation exposes it | Never log secrets |
| Kubernetes Secret (no encryption at rest) | etcd dump exposes it | Enable Encryption at Rest + use Vault |

---

## Cleanup

```bash
kubectl delete namespace secrets-demo
rm -rf ~/devsecops-demo/demo-09
```

---

## Key Takeaways

1. **Kubernetes Secrets are NOT encrypted** — they are base64-encoded. Anyone with `kubectl get secret` can read them.
2. **Never commit secrets to Git** — even if deleted, they persist in history.
3. **Volume mounts over env vars** for sensitive secrets — tighter permissions, no leak to child processes.
4. **Encryption at Rest** protects secrets if etcd is compromised — should be enabled in production.
5. For production, use an **external secret manager** (Vault, AWS Secrets Manager) instead of Kubernetes Secrets — see Demo 10.
