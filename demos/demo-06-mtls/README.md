# Demo 06: Mutual TLS (mTLS) Between Services

> **Duration:** ~10 minutes  
> **Section:** 3.3 — Service Communication Security, East-West Traffic Protection  
> **Tools:** minikube, kubectl, Helm, cert-manager

---

## Learning Objectives

- Understand the difference between one-way TLS and mutual TLS (mTLS)
- Install cert-manager to automate certificate issuance
- Deploy two services that authenticate each other with certificates
- Verify that connections without a valid client certificate are rejected

---

## Prerequisites

- Default minikube cluster running: `minikube status`
- Helm installed: `helm version`
- If needed: `minikube start --cpus=2 --memory=4096 --driver=docker`

---

## Background

| Mode | What's verified |
|------|----------------|
| **No TLS** | Nothing — plaintext traffic |
| **One-way TLS** | Client verifies server identity (HTTPS) |
| **Mutual TLS (mTLS)** | Both client and server verify each other's identity |

mTLS ensures that even if an attacker intercepts internal cluster traffic, they cannot impersonate a service without its certificate.

---

## Setup

```bash
mkdir -p ~/devsecops-demo/demo-06 && cd ~/devsecops-demo/demo-06
```

---

## Part 1: Install cert-manager

### Step 1 — Install cert-manager via Helm

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

# Wait for cert-manager pods to be ready
kubectl wait --for=condition=Ready pods --all -n cert-manager --timeout=120s
echo "cert-manager ready."
```

---

## Part 2: Create a Certificate Authority (CA)

### Step 2 — Create a self-signed CA for our internal services

```bash
cat > ca-issuer.yaml << 'EOF'
# Step 1: Bootstrap — create a self-signed issuer to sign our CA certificate
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-bootstrap
spec:
  selfSigned: {}
---
# Step 2: Create a CA certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: "internal-ca"
  subject:
    organizations:
      - "DevSecOps Training"
  secretName: internal-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-bootstrap
    kind: ClusterIssuer
---
# Step 3: Create a CA Issuer backed by our certificate
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca-issuer
spec:
  ca:
    secretName: internal-ca-secret
EOF

kubectl apply -f ca-issuer.yaml
sleep 5
kubectl get clusterissuer
```

**Expected output:**
```
NAME                  READY   AGE
internal-ca-issuer    True    5s
selfsigned-bootstrap  True    5s
```

---

## Part 3: Issue Service Certificates

### Step 3 — Create a namespace and issue certificates for two services

```bash
kubectl create namespace mtls-demo

cat > service-certs.yaml << 'EOF'
# Certificate for the "server" service
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: server-cert
  namespace: mtls-demo
spec:
  secretName: server-tls
  commonName: "server-svc.mtls-demo.svc.cluster.local"
  dnsNames:
    - "server-svc"
    - "server-svc.mtls-demo"
    - "server-svc.mtls-demo.svc.cluster.local"
  issuerRef:
    name: internal-ca-issuer
    kind: ClusterIssuer
  usages:
    - server auth
    - client auth
---
# Certificate for the "client" service
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: client-cert
  namespace: mtls-demo
spec:
  secretName: client-tls
  commonName: "client.mtls-demo.svc.cluster.local"
  dnsNames:
    - "client.mtls-demo.svc.cluster.local"
  issuerRef:
    name: internal-ca-issuer
    kind: ClusterIssuer
  usages:
    - client auth
    - server auth
EOF

kubectl apply -f service-certs.yaml

# Wait for certificates to be issued
kubectl wait --for=condition=Ready certificate/server-cert certificate/client-cert \
  -n mtls-demo --timeout=60s
echo "Certificates issued."
```

### Step 4 — Inspect the issued certificate

```bash
kubectl describe certificate server-cert -n mtls-demo | grep -A5 "Status:"
kubectl get secret server-tls -n mtls-demo -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -subject -issuer -dates
```

**Expected output:**
```
subject=CN = server-svc.mtls-demo.svc.cluster.local
issuer=CN = internal-ca
notBefore=...
notAfter=... (90 days from now)
```

---

## Part 4: Deploy Services with mTLS

### Step 5 — Deploy an nginx server with TLS configuration

```bash
cat > nginx-mtls.conf << 'EOF'
server {
    listen 443 ssl;
    server_name server-svc;

    ssl_certificate     /etc/nginx/tls/tls.crt;
    ssl_certificate_key /etc/nginx/tls/tls.key;
    ssl_client_certificate /etc/nginx/tls/ca.crt;
    ssl_verify_client on;       # This enforces mutual TLS

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        return 200 "Hello from mTLS-protected server!\n";
        add_header Content-Type text/plain;
    }
}
EOF

kubectl create configmap nginx-mtls-config \
  --from-file=default.conf=nginx-mtls.conf \
  -n mtls-demo
```

```bash
cat > server-deploy.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server
  namespace: mtls-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: server
  template:
    metadata:
      labels:
        app: server
    spec:
      containers:
      - name: server
        image: nginx:1.25-alpine
        ports:
        - containerPort: 443
        volumeMounts:
        - name: tls
          mountPath: /etc/nginx/tls
          readOnly: true
        - name: config
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: tls
        secret:
          secretName: server-tls
          items:
          - key: tls.crt
            path: tls.crt
          - key: tls.key
            path: tls.key
          - key: ca.crt
            path: ca.crt
      - name: config
        configMap:
          name: nginx-mtls-config
---
apiVersion: v1
kind: Service
metadata:
  name: server-svc
  namespace: mtls-demo
spec:
  selector:
    app: server
  ports:
  - port: 443
    targetPort: 443
EOF

kubectl apply -f server-deploy.yaml
kubectl wait --for=condition=Available deployment/server -n mtls-demo --timeout=60s
echo "Server deployed."
```

---

## Part 5: Verify mTLS Enforcement

### Step 6 — Deploy a client pod to test connections

```bash
kubectl run client \
  --image=alpine:3.19 \
  --namespace=mtls-demo \
  --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"client","image":"alpine:3.19","command":["sleep","3600"],"volumeMounts":[{"name":"tls","mountPath":"/etc/tls","readOnly":true}]}],"volumes":[{"name":"tls","secret":{"secretName":"client-tls"}}]}}' 

kubectl wait --for=condition=Ready pod/client -n mtls-demo --timeout=60s

# Install curl in the client pod
kubectl exec -n mtls-demo client -- apk add --no-cache curl -q
```

### Step 7 — Test WITHOUT a client certificate (should fail)

```bash
echo "=== Connecting WITHOUT client certificate (one-way TLS only) ==="
kubectl exec -n mtls-demo client -- \
  curl -sk https://server-svc/
```

**Expected output:**
```
<html>
<head><title>400 No required SSL certificate was sent</title></head>
...
```

The server **rejected the connection** because no client certificate was presented — mTLS enforcement works.

### Step 8 — Test WITH a client certificate (should succeed)

```bash
echo "=== Connecting WITH client certificate (mutual TLS) ==="
kubectl exec -n mtls-demo client -- \
  curl -sk \
  --cert /etc/tls/tls.crt \
  --key /etc/tls/tls.key \
  --cacert /etc/tls/ca.crt \
  https://server-svc/
```

**Expected output:**
```
Hello from mTLS-protected server!
```

Connection succeeds because the client presented a valid certificate signed by the trusted CA.

---

## mTLS in Production: Service Meshes

Implementing mTLS manually (as shown above) does not scale. In production, a **service mesh** automates this:

| Tool | Approach |
|------|---------|
| **Istio** | Sidecar proxy (Envoy) handles mTLS transparently |
| **Linkerd** | Lightweight sidecar, mTLS by default |
| **Cilium** | eBPF-based, no sidecar needed |

With a service mesh, developers write normal code — the mesh handles all certificate management and mTLS enforcement automatically.

---

## Cleanup

```bash
kubectl delete namespace mtls-demo
kubectl delete clusterissuer selfsigned-bootstrap internal-ca-issuer
helm uninstall cert-manager -n cert-manager
kubectl delete namespace cert-manager
rm -rf ~/devsecops-demo/demo-06
```

---

## Key Takeaways

1. **mTLS means both parties prove their identity** — not just the server to the client.
2. `ssl_verify_client on` in nginx (or equivalent) is what enforces the "mutual" part.
3. **cert-manager automates** certificate issuance, renewal, and rotation — manual certificate management doesn't scale.
4. For production workloads, use a **service mesh** (Istio, Linkerd, Cilium) to get mTLS without modifying application code.
5. mTLS protects **east-west traffic** (service-to-service inside the cluster) — a critical attack path often overlooked.
