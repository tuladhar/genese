# DevSecOps Workshop — Live Demo Guide

**Client:** Dangote Group, Nigeria  
**Duration:** ~2 hours  
**Topic:** Container Security · Kubernetes Security · Secrets Management

---

## Agenda

| # | Topic | Time |
|---|-------|------|
| Setup | Environment Check & Minikube Start | 10 min |
| Demo 01 | Docker Image Hardening | 10 min |
| Demo 02 | Trivy Vulnerability Scanning & SBOM | 8 min |
| Demo 03 | Pod Hardening & Pod Security Admission | 10 min |
| Demo 04 | Namespace & Node Isolation | 8 min |
| Demo 05 | Network Policy with Calico | 10 min |
| Demo 06 | Mutual TLS (mTLS) Between Services | 10 min |
| Demo 07 | RBAC — Least Privilege | 8 min |
| Demo 08 | Kubernetes Audit Logging | 8 min |
| Demo 09 | Kubernetes Secrets | 8 min |
| Demo 10 | HashiCorp Vault + External Secrets Operator | 12 min |
| Demo 11 | Secret Scanning with Gitleaks | 6 min |
| Demo 12 | Runtime Threat Detection with Falco | 10 min |

> Detailed explanations and YAML references are in each `demo-XX-*/README.md` directory.

---

## Before You Start

### Tools required

| Tool | Install (Windows) | Install (Ubuntu) |
|------|-------------------|------------------|
| Docker Desktop | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) | `sudo apt install docker.io` |
| minikube | `winget install Kubernetes.minikube` | Download from storage.googleapis.com |
| kubectl | `winget install Kubernetes.kubectl` | Download from dl.k8s.io |
| Helm | `winget install Helm.Helm` | `curl ...get-helm-3 \| bash` |
| Trivy | `scoop install trivy` | Add aquasecurity APT repo |
| Gitleaks | `winget install GitLeaks.GitLeaks` | Download binary from GitHub releases |

Full installation instructions: [00-prerequisites.md](./00-prerequisites/)

### Environment check

```bash
docker --version
minikube version
kubectl version --client
helm version --short
trivy --version
gitleaks version
```

### Start minikube (used by most demos)

```bash
minikube start --cpus=2 --memory=4096 --driver=docker
kubectl get nodes
```

**Expected:**
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   30s   v1.31.0
```

### Pre-pull images (saves time during session)

```bash
docker pull python:3.8
docker pull python:3.8-alpine
docker pull python:3.11-alpine
docker pull nginx:1.25-alpine
docker pull busybox:1.36
docker pull ubuntu:22.04
```

---

---

## Demo 01 — Docker Image Hardening

> **10 min · Docker only · no Kubernetes needed**  
> Show: bloated root-running image vs minimal non-root hardened image

### Setup

```bash
mkdir -p ~/devsecops-demo/demo-01 && cd ~/devsecops-demo/demo-01
```

### Step 1 — The insecure Dockerfile

```bash
cat > Dockerfile.insecure << 'EOF'
FROM python:3.8

COPY app.py .

RUN apt-get update && apt-get install -y curl vim

EXPOSE 8080
CMD ["python", "app.py"]
EOF
```

### Step 2 — The sample app

```bash
cat > app.py << 'EOF'
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hello from DevSecOps Demo!")
    def log_message(self, format, *args):
        pass

HTTPServer(("", 8080), Handler).serve_forever()
EOF
```

### Step 3 — Build the insecure image and inspect it

```bash
docker build -f Dockerfile.insecure -t demo-app:insecure .
docker images demo-app:insecure
docker run --rm demo-app:insecure whoami
```

**Expected:**
```
REPOSITORY   TAG        SIZE
demo-app     insecure   921MB

root
```

> **Point out:** Nearly 1 GB, runs as root. A compromised process has full container access.

### Step 4 — Scan for CVEs

```bash
trivy image --severity HIGH,CRITICAL demo-app:insecure 2>/dev/null | tail -5
```

> Dozens of HIGH/CRITICAL CVEs from the bloated Debian base.

### Step 5 — The hardened Dockerfile (multi-stage, non-root, Alpine)

```bash
cat > Dockerfile.hardened << 'EOF'
FROM python:3.8-alpine AS builder
WORKDIR /app
COPY app.py .

FROM python:3.8-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/app.py .
RUN chown -R appuser:appgroup /app
USER appuser
EXPOSE 8080
CMD ["python", "app.py"]
EOF

docker build -f Dockerfile.hardened -t demo-app:hardened .
```

### Step 6 — Compare sizes and verify non-root

```bash
docker images demo-app
docker run --rm demo-app:hardened whoami
```

**Expected:**
```
REPOSITORY   TAG       SIZE
demo-app     hardened  49.2MB
demo-app     insecure  921MB

appuser
```

### Step 7 — Run with runtime hardening flags

```bash
docker run -d \
  --name demo-hardened \
  --read-only \
  --cap-drop=ALL \
  --security-opt no-new-privileges:true \
  --tmpfs /tmp \
  -p 8888:8080 \
  demo-app:hardened

curl http://localhost:8888
```

**Expected:** `Hello from DevSecOps Demo!`

### Step 8 — Prove read-only filesystem

```bash
docker exec demo-hardened sh -c "echo test > /app/test.txt" 2>&1
```

**Expected:** `sh: can't create /app/test.txt: Read-only file system`

### Step 9 — Scan the hardened image

```bash
trivy image --severity HIGH,CRITICAL demo-app:hardened 2>/dev/null | tail -5
```

> Dramatically fewer CVEs — Alpine's minimal footprint.

### Cleanup

```bash
docker stop demo-hardened && docker rm demo-hardened
docker rmi demo-app:insecure demo-app:hardened
rm -rf ~/devsecops-demo/demo-01
```

**Key takeaways:**
- Minimal base images (Alpine) cut attack surface by ~95%
- Non-root user, `--read-only`, `--cap-drop=ALL` are the three most important runtime flags
- Multi-stage builds keep build tools out of production images

---

---

## Demo 02 — Trivy Vulnerability Scanning & SBOM

> **8 min · Trivy only · no Kubernetes needed**  
> Show: CVE scanning, SBOM generation, CI gate, Dockerfile scanning

### Setup

```bash
mkdir -p ~/devsecops-demo/demo-02 && cd ~/devsecops-demo/demo-02
```

### Step 1 — Scan a known-vulnerable image

```bash
trivy image python:3.8 --severity HIGH,CRITICAL
```

> First run downloads the vuln DB (~30s). Shows 1600+ HIGH/CRITICAL CVEs.

### Step 2 — Scan a minimal image for comparison

```bash
trivy image python:3.11-alpine --severity HIGH,CRITICAL
```

**Expected:** `Total: 0 (HIGH: 0, CRITICAL: 0)` for OS packages.

### Step 3 — CI/CD gate: fail on CRITICAL CVEs

```bash
trivy image --exit-code 1 --severity CRITICAL python:3.8
echo "Exit code: $?"
```

**Expected:** `Exit code: 1`  
> `--exit-code 1` makes Trivy exit non-zero — this fails a CI pipeline build.

### Step 4 — Generate an SBOM (Software Bill of Materials)

```bash
trivy image --format spdx-json --output python-3.8-sbom.spdx.json python:3.8
echo "SBOM saved."
wc -l python-3.8-sbom.spdx.json
```

### Step 5 — Scan an existing SBOM (re-scan without the original image)

```bash
trivy sbom python-3.8-sbom.spdx.json --severity HIGH,CRITICAL
```

> **Point out:** When a new CVE drops, re-scan the SBOM — no need to rebuild the image.

### Step 6 — Scan a Dockerfile for misconfigurations

```bash
cat > Dockerfile.scan-test << 'EOF'
FROM ubuntu:latest
RUN apt-get update && apt-get install -y curl
ENV SECRET_KEY=hardcoded_secret_123
USER root
CMD ["/bin/bash"]
EOF

trivy config Dockerfile.scan-test
```

> Trivy flags: `latest` tag, running as root, hardcoded secret in ENV.

### Step 7 — Scan a Kubernetes manifest

```bash
cat > k8s-insecure.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
spec:
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      privileged: true
      runAsUser: 0
EOF

trivy config k8s-insecure.yaml
```

**Expected highlights:**
```
HIGH: Container 'app' should not run as root
HIGH: Container 'app' should not be privileged
```

### Cleanup

```bash
rm -rf ~/devsecops-demo/demo-02
```

**Key takeaways:**
- SBOM = inventory of your software — generate at build time, re-scan on new CVE disclosures
- `--exit-code 1` is the CI security gate — blocks deployments with critical vulnerabilities
- Trivy scans images, SBOMs, Dockerfiles, Kubernetes manifests, and git repos

---

---

## Demo 03 — Pod Hardening & Pod Security Admission

> **10 min · minikube + kubectl**  
> Show: insecure pod, PSA blocks it, hardened pod passes

### Setup

```bash
mkdir -p ~/devsecops-demo/demo-03 && cd ~/devsecops-demo/demo-03
```

### Step 1 — Deploy an insecure pod

```bash
cat > pod-insecure.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
spec:
  containers:
  - name: app
    image: nginx:1.25-alpine
    securityContext:
      privileged: true
      runAsUser: 0
      allowPrivilegeEscalation: true
    resources: {}
EOF

kubectl apply -f pod-insecure.yaml
kubectl wait --for=condition=Ready pod/insecure-pod --timeout=60s
```

### Step 2 — Examine what's running

```bash
kubectl exec insecure-pod -- id
kubectl exec insecure-pod -- sh -c "echo 'pwned' > /tmp/test && cat /tmp/test"
kubectl exec insecure-pod -- cat /proc/1/status | grep -E "CapEff|CapPrm"
```

**Expected:** `uid=0(root)` — full capabilities bitmask.

### Step 3 — Create a restricted namespace

```bash
kubectl create namespace secure-ns

kubectl label namespace secure-ns \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest
```

### Step 4 — Try deploying the insecure pod into the restricted namespace

```bash
kubectl apply -f pod-insecure.yaml -n secure-ns
```

**Expected:**
```
Error from server (Forbidden): pods "insecure-pod" is forbidden: violates PodSecurity "restricted:latest":
  privileged, runAsNonRoot, allowPrivilegeEscalation ...
```

> **Point out:** Pod never starts. Admission controller blocked it before any code ran.

### Step 5 — Deploy a properly hardened pod

```bash
cat > pod-hardened.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
  namespace: secure-ns
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.25-alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: [ALL]
    resources:
      requests: {cpu: "100m", memory: "64Mi"}
      limits: {cpu: "200m", memory: "128Mi"}
    volumeMounts:
    - {name: tmp, mountPath: /tmp}
    - {name: cache, mountPath: /var/cache/nginx}
    - {name: run, mountPath: /var/run}
  volumes:
  - {name: tmp, emptyDir: {}}
  - {name: cache, emptyDir: {}}
  - {name: run, emptyDir: {}}
EOF

kubectl apply -f pod-hardened.yaml
kubectl wait --for=condition=Ready pod/hardened-pod -n secure-ns --timeout=60s
```

### Step 6 — Verify hardening

```bash
kubectl exec -n secure-ns hardened-pod -- id
kubectl exec -n secure-ns hardened-pod -- sh -c "echo test > /etc/test.txt" 2>&1
kubectl exec -n secure-ns hardened-pod -- sh -c "echo 'tmp works' > /tmp/test && cat /tmp/test"
```

**Expected:**
```
uid=1000 gid=1000 groups=1000
sh: can't create /etc/test.txt: Read-only file system
tmp works
```

### Cleanup

```bash
kubectl delete pod insecure-pod
kubectl delete pod hardened-pod -n secure-ns
kubectl delete namespace secure-ns
rm -rf ~/devsecops-demo/demo-03
```

**Key takeaways:**
- Pod Security Admission is built into Kubernetes — label namespaces to enforce
- `runAsNonRoot`, `readOnlyRootFilesystem`, `capabilities: drop: ALL` are the three key settings
- PSA blocks at admission — before any code runs

---

---

## Demo 04 — Namespace & Node Isolation

> **8 min · minikube + kubectl**  
> Show: different PSA levels per namespace, RBAC namespace scoping, taints

### Setup

```bash
mkdir -p ~/devsecops-demo/demo-04 && cd ~/devsecops-demo/demo-04
```

### Step 1 — Create environment namespaces

```bash
kubectl create namespace dev 2>/dev/null || kubectl label namespace dev app=demo --overwrite
kubectl create namespace staging 2>/dev/null || kubectl label namespace staging app=demo --overwrite
kubectl create namespace production 2>/dev/null || kubectl label namespace production app=demo --overwrite
```

### Step 2 — Apply different security levels per namespace

```bash
# dev: warn only — pods still run, but warnings appear
kubectl label namespace dev \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=latest

# staging: enforce baseline — blocks known privilege escalations
kubectl label namespace staging \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest

# production: enforce restricted — strictest
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest
```

### Step 3 — Test the same root-user pod across namespaces

```bash
cat > test-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 0
      privileged: false
EOF

kubectl apply -f test-pod.yaml -n dev && echo "dev: allowed"
kubectl apply -f test-pod.yaml -n staging && echo "staging: allowed"
kubectl apply -f test-pod.yaml -n production 2>&1 | head -3
```

**Expected:**
```
dev: allowed (with warning)
staging: allowed
Error from server (Forbidden): violates PodSecurity "restricted:latest" ...
```

### Step 4 — RBAC: dev-user can only see the dev namespace

```bash
kubectl create serviceaccount dev-user -n dev

cat > dev-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-access
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-access-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: dev-access
EOF

kubectl apply -f dev-role.yaml
```

```bash
# Can dev-user see pods in dev?
kubectl auth can-i list pods -n dev --as=system:serviceaccount:dev:dev-user

# Can dev-user see pods in production?
kubectl auth can-i list pods -n production --as=system:serviceaccount:dev:dev-user
```

**Expected:** `yes` / `no`

### Step 5 — Taint node: only workloads with a toleration can schedule

```bash
kubectl taint nodes minikube workload-type=secure:NoSchedule

# Pod without toleration stays Pending
cat > pod-no-toleration.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-no-toleration
  namespace: dev
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
EOF

kubectl apply -f pod-no-toleration.yaml
sleep 5
kubectl get pod pod-no-toleration -n dev
```

**Expected:** `Pending` — `0/1 nodes available: taint {workload-type: secure}`

```bash
# Pod WITH toleration runs
cat > pod-with-toleration.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-toleration
  namespace: dev
spec:
  tolerations:
  - key: "workload-type"
    operator: "Equal"
    value: "secure"
    effect: "NoSchedule"
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
EOF

kubectl apply -f pod-with-toleration.yaml
kubectl wait --for=condition=Ready pod/pod-with-toleration -n dev --timeout=30s
kubectl get pod pod-with-toleration -n dev
```

**Expected:** `Running`

### Cleanup

```bash
kubectl taint nodes minikube workload-type=secure:NoSchedule-
kubectl delete pod pod-no-toleration pod-with-toleration -n dev --ignore-not-found
kubectl delete pod test-pod -n dev --ignore-not-found
kubectl delete pod test-pod -n staging --ignore-not-found
kubectl delete namespace dev staging production
rm -rf ~/devsecops-demo/demo-04
```

**Key takeaways:**
- Namespaces alone are not security boundaries — add PSA labels and RBAC
- Apply `restricted` to production, looser in dev for developer experience
- Taints prevent sensitive workloads from landing on general-purpose nodes

---

---

## Demo 05 — Network Policy with Calico

> **10 min · requires a separate minikube cluster**  
> Show: default allow-all, then default-deny, then selective allow

### Start a Calico-enabled cluster

```bash
minikube start \
  --profile=netpolicy \
  --cpus=2 \
  --memory=4096 \
  --driver=docker \
  --cni=calico

kubectl config use-context netpolicy
kubectl wait --for=condition=Ready pods --all -n kube-system --timeout=180s
echo "Cluster ready."
```

### Setup

```bash
mkdir -p ~/devsecops-demo/demo-05 && cd ~/devsecops-demo/demo-05
kubectl create namespace netdemo
```

### Step 1 — Deploy frontend / backend / database

```bash
cat > apps.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: netdemo
spec:
  replicas: 1
  selector:
    matchLabels: {app: frontend, tier: frontend}
  template:
    metadata:
      labels: {app: frontend, tier: frontend}
    spec:
      containers:
      - name: frontend
        image: nginx:1.25-alpine
        ports: [{containerPort: 80}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: netdemo
spec:
  replicas: 1
  selector:
    matchLabels: {app: backend, tier: backend}
  template:
    metadata:
      labels: {app: backend, tier: backend}
    spec:
      containers:
      - name: backend
        image: nginx:1.25-alpine
        ports: [{containerPort: 80}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: netdemo
spec:
  replicas: 1
  selector:
    matchLabels: {app: database, tier: database}
  template:
    metadata:
      labels: {app: database, tier: database}
    spec:
      containers:
      - name: database
        image: nginx:1.25-alpine
        ports: [{containerPort: 80}]
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: netdemo
spec:
  selector: {app: backend}
  ports: [{port: 80}]
---
apiVersion: v1
kind: Service
metadata:
  name: database-svc
  namespace: netdemo
spec:
  selector: {app: database}
  ports: [{port: 80}]
EOF

kubectl apply -f apps.yaml
kubectl wait --for=condition=Available deployment/frontend deployment/backend deployment/database \
  -n netdemo --timeout=90s
echo "All pods ready."
```

### Step 2 — Show default: all traffic allowed (security problem)

```bash
FRONTEND_POD=$(kubectl get pod -n netdemo -l app=frontend -o jsonpath='{.items[0].metadata.name}')

echo "=== Frontend → Backend (allowed) ==="
kubectl exec -n netdemo $FRONTEND_POD -- wget -qO- --timeout=3 http://backend-svc 2>&1 && echo "CONNECTED"

echo "=== Frontend → Database (allowed — but this is a problem!) ==="
kubectl exec -n netdemo $FRONTEND_POD -- wget -qO- --timeout=3 http://database-svc 2>&1 && echo "CONNECTED"
```

**Expected:** Both `CONNECTED`. Frontend can reach the database directly — lateral movement risk.

### Step 3 — Apply default-deny

```bash
cat > default-deny.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: netdemo
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF

kubectl apply -f default-deny.yaml
echo "Default deny applied."
```

```bash
echo "=== Frontend → Backend (should FAIL) ==="
kubectl exec -n netdemo $FRONTEND_POD -- wget -qO- --timeout=3 http://backend-svc 2>&1 && echo "CONNECTED" || echo "BLOCKED"
```

**Expected:** `BLOCKED`

### Step 4 — Selectively allow frontend → backend only

```bash
cat > allow-frontend-backend.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: netdemo
spec:
  podSelector:
    matchLabels: {tier: frontend}
  policyTypes: [Egress]
  egress:
  - to:
    - podSelector:
        matchLabels: {tier: backend}
    ports: [{protocol: TCP, port: 80}]
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports: [{protocol: UDP, port: 53}]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-ingress-from-frontend
  namespace: netdemo
spec:
  podSelector:
    matchLabels: {tier: backend}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {tier: frontend}
    ports: [{protocol: TCP, port: 80}]
EOF

kubectl apply -f allow-frontend-backend.yaml
```

```bash
cat > allow-backend-database.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: netdemo
spec:
  podSelector:
    matchLabels: {tier: backend}
  policyTypes: [Egress]
  egress:
  - to:
    - podSelector:
        matchLabels: {tier: database}
    ports: [{protocol: TCP, port: 80}]
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports: [{protocol: UDP, port: 53}]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-ingress-from-backend
  namespace: netdemo
spec:
  podSelector:
    matchLabels: {tier: database}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {tier: backend}
    ports: [{protocol: TCP, port: 80}]
EOF

kubectl apply -f allow-backend-database.yaml
```

### Step 5 — Verify the traffic matrix

```bash
BACKEND_POD=$(kubectl get pod -n netdemo -l app=backend -o jsonpath='{.items[0].metadata.name}')

echo "=== Frontend → Backend (should SUCCEED) ==="
kubectl exec -n netdemo $FRONTEND_POD -- wget -qO- --timeout=5 http://backend-svc 2>&1 && echo "CONNECTED" || echo "BLOCKED"

echo "=== Frontend → Database (should FAIL) ==="
kubectl exec -n netdemo $FRONTEND_POD -- wget -qO- --timeout=5 http://database-svc 2>&1 && echo "CONNECTED" || echo "BLOCKED"

echo "=== Backend → Database (should SUCCEED) ==="
kubectl exec -n netdemo $BACKEND_POD -- wget -qO- --timeout=5 http://database-svc 2>&1 && echo "CONNECTED" || echo "BLOCKED"
```

**Expected:**
```
Frontend → Backend:   CONNECTED
Frontend → Database:  BLOCKED
Backend → Database:   CONNECTED
```

### Cleanup

```bash
kubectl delete namespace netdemo
rm -rf ~/devsecops-demo/demo-05
minikube stop --profile=netpolicy
kubectl config use-context minikube
```

**Key takeaways:**
- Kubernetes allows all pod-to-pod traffic by default — a compromised frontend can reach your database
- NetworkPolicy requires a CNI that enforces it — Calico, Cilium, Weave. Default minikube CNI does not enforce
- Start with default-deny-all, then add selective allow rules
- DNS egress (UDP 53 to kube-system) must be explicitly allowed when using default-deny-egress

---

---

## Demo 06 — Mutual TLS (mTLS) Between Services

> **10 min · minikube + Helm + cert-manager**  
> Show: 400 without client cert → success with cert

### Setup

```bash
mkdir -p ~/devsecops-demo/demo-06 && cd ~/devsecops-demo/demo-06
kubectl create namespace mtls-demo
```

### Step 1 — Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --wait

echo "cert-manager ready."
```

### Step 2 — Create a self-signed CA and issue certificates

```bash
cat > pki.yaml << 'EOF'
# Self-signed bootstrap issuer
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-bootstrap
spec:
  selfSigned: {}
---
# CA certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-ca
  namespace: mtls-demo
spec:
  isCA: true
  commonName: internal-ca
  secretName: internal-ca-tls
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-bootstrap
    kind: ClusterIssuer
---
# CA-backed issuer for signing service certs
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: internal-ca-issuer
  namespace: mtls-demo
spec:
  ca:
    secretName: internal-ca-tls
---
# Server certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: server-cert
  namespace: mtls-demo
spec:
  secretName: server-tls
  commonName: mtls-server.mtls-demo.svc.cluster.local
  dnsNames:
    - mtls-server
    - mtls-server.mtls-demo.svc.cluster.local
  issuerRef:
    name: internal-ca-issuer
    kind: Issuer
---
# Client certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: client-cert
  namespace: mtls-demo
spec:
  secretName: client-tls
  commonName: mtls-client
  usages: [client auth]
  issuerRef:
    name: internal-ca-issuer
    kind: Issuer
EOF

kubectl apply -f pki.yaml
sleep 10
kubectl get certificates -n mtls-demo
```

**Expected:** `READY=True` for both certificates.

### Step 3 — Deploy the mTLS-protected nginx server

```bash
cat > mtls-server.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-mtls-config
  namespace: mtls-demo
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 443 ssl;
        ssl_certificate     /etc/nginx/certs/tls.crt;
        ssl_certificate_key /etc/nginx/certs/tls.key;
        ssl_client_certificate /etc/nginx/ca/tls.crt;
        ssl_verify_client on;
        location / {
          return 200 "Hello from mTLS-protected server!\n";
          add_header Content-Type text/plain;
        }
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: mtls-server
  namespace: mtls-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.25-alpine
    ports: [{containerPort: 443}]
    volumeMounts:
    - name: server-tls
      mountPath: /etc/nginx/certs
      readOnly: true
    - name: ca-cert
      mountPath: /etc/nginx/ca
      readOnly: true
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
  volumes:
  - name: server-tls
    secret: {secretName: server-tls}
  - name: ca-cert
    secret: {secretName: internal-ca-tls}
  - name: nginx-config
    configMap: {name: nginx-mtls-config}
---
apiVersion: v1
kind: Service
metadata:
  name: mtls-server
  namespace: mtls-demo
spec:
  selector: {name: mtls-server}
  ports: [{port: 443, targetPort: 443}]
EOF

kubectl apply -f mtls-server.yaml
kubectl wait --for=condition=Ready pod/mtls-server -n mtls-demo --timeout=60s
```

### Step 4 — Test WITHOUT client cert (should fail)

```bash
kubectl run curl-test --image=curlimages/curl:8.5.0 -n mtls-demo --restart=Never \
  --command -- sleep 3600
kubectl wait --for=condition=Ready pod/curl-test -n mtls-demo --timeout=30s

kubectl exec -n mtls-demo curl-test -- \
  curl -sk https://mtls-server.mtls-demo.svc.cluster.local/ 2>&1
```

**Expected:** `400 Bad Request` — no required SSL certificate.

### Step 5 — Test WITH client cert (should succeed)

```bash
# Copy the client certs into the curl pod
kubectl exec -n mtls-demo curl-test -- mkdir -p /tmp/certs
kubectl cp mtls-demo/$(kubectl get pod -n mtls-demo -l app=mtls-server -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || echo mtls-server)/etc/nginx/certs/tls.crt /tmp/server.crt 2>/dev/null || true

# Extract client certs directly from the secret
kubectl get secret client-tls -n mtls-demo -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/client.crt
kubectl get secret client-tls -n mtls-demo -o jsonpath='{.data.tls\.key}' | base64 -d > /tmp/client.key
kubectl get secret internal-ca-tls -n mtls-demo -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/ca.crt

kubectl cp /tmp/client.crt mtls-demo/curl-test:/tmp/client.crt
kubectl cp /tmp/client.key mtls-demo/curl-test:/tmp/client.key
kubectl cp /tmp/ca.crt mtls-demo/curl-test:/tmp/ca.crt

kubectl exec -n mtls-demo curl-test -- \
  curl -s \
    --cacert /tmp/ca.crt \
    --cert /tmp/client.crt \
    --key /tmp/client.key \
    https://mtls-server.mtls-demo.svc.cluster.local/
```

**Expected:** `Hello from mTLS-protected server!`

### Cleanup

```bash
kubectl delete namespace mtls-demo cert-manager
helm uninstall cert-manager -n cert-manager 2>/dev/null || true
rm -rf ~/devsecops-demo/demo-06 /tmp/client.crt /tmp/client.key /tmp/ca.crt
```

**Key takeaways:**
- mTLS authenticates both sides of the connection — server proves identity to client AND client proves identity to server
- cert-manager automates certificate issuance and renewal
- Without a valid client cert, the server returns 400 — no code needed, nginx enforces it at the TLS layer
- In production, use a service mesh (Istio, Linkerd) to handle mTLS automatically for all services

---

---

## Demo 07 — RBAC: Least Privilege

> **8 min · minikube + kubectl**  
> Show: over-privileged SA vs minimal SA, audit who has cluster-admin

### Setup

```bash
mkdir -p ~/devsecops-demo/demo-07 && cd ~/devsecops-demo/demo-07
kubectl create namespace rbac-demo
```

### Step 1 — Create an over-privileged service account (bad practice)

```bash
kubectl create serviceaccount over-privileged-sa -n rbac-demo

kubectl create clusterrolebinding over-privileged-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=rbac-demo:over-privileged-sa
```

### Step 2 — Show what it can do

```bash
SA="system:serviceaccount:rbac-demo:over-privileged-sa"

echo "List secrets in kube-system?"
kubectl auth can-i list secrets -n kube-system --as=$SA

echo "Delete nodes?"
kubectl auth can-i delete nodes --as=$SA

echo "Create ClusterRoleBindings?"
kubectl auth can-i create clusterrolebindings --as=$SA
```

**Expected:** `yes` to all. Full cluster control — one compromised pod = game over.

### Step 3 — Create a least-privilege service account

```bash
kubectl create serviceaccount backend-api-sa -n rbac-demo

cat > backend-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-api-role
  namespace: rbac-demo
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-api-binding
  namespace: rbac-demo
subjects:
- kind: ServiceAccount
  name: backend-api-sa
  namespace: rbac-demo
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: backend-api-role
EOF

kubectl apply -f backend-role.yaml
```

### Step 4 — Verify least-privilege

```bash
SA_BACKEND="system:serviceaccount:rbac-demo:backend-api-sa"

echo "=== Allowed ==="
kubectl auth can-i list configmaps -n rbac-demo --as=$SA_BACKEND
kubectl auth can-i get secrets -n rbac-demo --as=$SA_BACKEND

echo "=== Blocked ==="
kubectl auth can-i delete pods -n rbac-demo --as=$SA_BACKEND
kubectl auth can-i list secrets -n kube-system --as=$SA_BACKEND
kubectl auth can-i create clusterrolebindings --as=$SA_BACKEND
```

**Expected:** `yes yes` / `no no no`

### Step 5 — Audit: who has cluster-admin?

```bash
kubectl get clusterrolebindings -o json | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
for item in data['items']:
    if item.get('roleRef', {}).get('name') == 'cluster-admin':
        for s in item.get('subjects', []):
            print(f\"  {s.get('kind')}/{s.get('namespace','cluster')}/{s.get('name')} via {item['metadata']['name']}\")
"
```

> **Point out:** Our `over-privileged-sa` shows up. This is how you audit a real cluster.

### Step 6 — List all permissions for a service account

```bash
kubectl auth can-i --list -n rbac-demo \
  --as=system:serviceaccount:rbac-demo:backend-api-sa
```

### Cleanup

```bash
kubectl delete namespace rbac-demo
kubectl delete clusterrolebinding over-privileged-binding
rm -rf ~/devsecops-demo/demo-07
```

**Key takeaways:**
- Never bind `cluster-admin` to application service accounts
- Use `Role` (namespace-scoped) not `ClusterRole` when possible
- `kubectl auth can-i --list` audits every permission for any service account

---

---

## Demo 08 — Kubernetes Audit Logging

> **8 min · minikube + kubectl**  
> Show: configure audit policy, restart minikube, read audit events

### Setup

```bash
mkdir -p ~/devsecops-demo/demo-08 && cd ~/devsecops-demo/demo-08
```

### Step 1 — Write the audit policy

```bash
cat > audit-policy.yaml << 'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["secrets"]

  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    resources:
    - group: ""
      resources: ["pods"]

  - level: RequestResponse
    resources:
    - group: "rbac.authorization.k8s.io"
      resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]

  - level: Metadata
    omitStages: [RequestReceived]
    users: ["system:anonymous"]

  - level: None
    nonResourceURLs: ["/healthz", "/readyz", "/livez", "/metrics"]

  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: ""
      resources: ["endpoints", "services", "services/status"]

  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get"]
    resources:
    - group: ""
      resources: ["nodes", "nodes/status"]

  - level: Metadata
    omitStages: [RequestReceived]
EOF
```

### Step 2 — Copy policy into minikube and restart with audit flags

```bash
minikube start --cpus=2 --memory=4096 --driver=docker
minikube ssh "sudo mkdir -p /etc/ssl/certs"
cat audit-policy.yaml | minikube ssh "sudo tee /etc/ssl/certs/audit-policy.yaml > /dev/null"
minikube ssh "sudo head -5 /etc/ssl/certs/audit-policy.yaml"
```

```bash
minikube stop

minikube start \
  --cpus=2 \
  --memory=4096 \
  --driver=docker \
  --extra-config=apiserver.audit-policy-file=/etc/ssl/certs/audit-policy.yaml \
  --extra-config=apiserver.audit-log-path=-

kubectl cluster-info
echo "Audit logging enabled."
```

> `--audit-log-path=-` writes audit events to the API server's stderr, readable via `kubectl logs`. This is the correct approach for minikube with the Docker driver.

### Step 3 — Generate audit events

```bash
kubectl create namespace audit-test

kubectl create secret generic db-password \
  --from-literal=password=SuperSecret123 \
  -n audit-test

kubectl get secret db-password -n audit-test -o yaml

kubectl create serviceaccount audit-sa -n audit-test

kubectl create rolebinding audit-test-binding \
  --clusterrole=view \
  --serviceaccount=audit-test:audit-sa \
  -n audit-test
```

### Step 4 — Read the audit log

```bash
echo "=== Audit Log Entries ==="
kubectl logs kube-apiserver-minikube -n kube-system --tail=500 2>/dev/null | \
  grep '"kind":"Event"' | \
  python3 -c "
import sys, json

for line in sys.stdin:
    line = line.strip()
    if not line:
        continue
    try:
        entry = json.loads(line)
        verb = entry.get('verb', '')
        resource = entry.get('objectRef', {}).get('resource', '')
        name = entry.get('objectRef', {}).get('name', '')
        namespace = entry.get('objectRef', {}).get('namespace', '')
        user = entry.get('user', {}).get('username', '')
        level = entry.get('level', '')
        status = entry.get('responseStatus', {}).get('code', '')

        if resource in ('secrets', 'rolebindings', 'clusterroles') or 'admin' in user:
            print(f'[{level}] {user} | {verb} {resource}/{name} (ns:{namespace}) -> HTTP {status}')
    except:
        pass
" 2>/dev/null | head -20
```

**Sample output:**
```
[RequestResponse] kubernetes-admin | create secrets/db-password (ns:audit-test) -> HTTP 201
[RequestResponse] kubernetes-admin | get secrets/db-password (ns:audit-test) -> HTTP 200
[RequestResponse] kubernetes-admin | create rolebindings/audit-test-binding (ns:audit-test) -> HTTP 201
```

### Step 5 — Show a full audit entry for a secret access

```bash
kubectl logs kube-apiserver-minikube -n kube-system --tail=200 2>/dev/null | \
  grep '"kind":"Event"' | \
  python3 -c "
import sys, json
for line in sys.stdin:
    try:
        entry = json.loads(line.strip())
        if entry.get('verb') == 'get' and \
           entry.get('objectRef', {}).get('resource') == 'secrets':
            print(json.dumps(entry, indent=2))
            break
    except:
        pass
" 2>/dev/null | head -30
```

### Cleanup

```bash
kubectl delete namespace audit-test
rm -rf ~/devsecops-demo/demo-08

minikube stop
minikube start --cpus=2 --memory=4096 --driver=docker
echo "Cluster restored."
```

**Key takeaways:**
- Audit logs answer "who did what, to which resource, and when?"
- Don't log everything at `RequestResponse` — secrets and RBAC changes yes, health checks never
- In production, ship audit logs to a SIEM — not just a file on the node
- Audit logging catches insider threats AND external attackers

---

---

## Demo 09 — Kubernetes Secrets

> **8 min · minikube + kubectl**  
> Show: base64 is not encryption, env var vs volume mount, encryption at rest

### Setup

```bash
mkdir -p ~/devsecops-demo/demo-09 && cd ~/devsecops-demo/demo-09
kubectl create namespace secrets-demo
```

### Step 1 — Base64 is NOT encryption

```bash
echo -n "SuperSecretPassword123" | base64
echo "U3VwZXJTZWNyZXRQYXNzd29yZDEyMw==" | base64 -d
echo ""
```

**Expected:** Instantly reversible. Anyone with `kubectl get secret` can decode it.

### Step 2 — Create a secret and show the exposure

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecretPassword123 \
  -n secrets-demo

kubectl get secret db-credentials -n secrets-demo -o yaml

kubectl get secret db-credentials -n secrets-demo \
  -o jsonpath='{.data.password}' | base64 -d
echo ""
```

### Step 3 — Show bad patterns (for discussion, don't run)

```bash
# BAD: password baked into image
cat > Dockerfile.bad-secret << 'EOF'
FROM alpine:3.19
ENV DB_PASSWORD=SuperSecretPassword123
EOF
echo "docker inspect on this image exposes the password."

# BAD: secret committed to git
echo "Even if deleted from HEAD, it stays in git history forever."
```

### Step 4 — Consume secrets as environment variables

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

**Expected:** `DB_USER=admin` / `DB_PASS is set: yes`

### Step 5 — Consume secrets as volume mounts (preferred)

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
    fsGroup: 1000
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Password file:' && cat /etc/db-creds/password && echo && sleep 3600"]
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
      defaultMode: 0440
  restartPolicy: Never
EOF

kubectl apply -f pod-volume-secret.yaml
kubectl wait --for=condition=Ready pod/pod-volume-secret -n secrets-demo --timeout=30s
kubectl logs pod/pod-volume-secret -n secrets-demo
kubectl exec -n secrets-demo pod/pod-volume-secret -- ls -la /etc/db-creds/
```

**Expected:** Password printed, file shows `r--r-----` permissions.

### Step 6 — Check if encryption at rest is enabled

```bash
minikube ssh "sudo ps aux | grep apiserver" | grep -o "encryption-provider-config[^ ]*" || \
  echo "Encryption at rest: NOT configured (minikube default)"
```

> In production: AES-CBC or KMS provider encrypts secrets in etcd.

### Cleanup

```bash
kubectl delete namespace secrets-demo
rm -rf ~/devsecops-demo/demo-09
```

**Key takeaways:**
- Kubernetes Secrets are base64-encoded, NOT encrypted — anyone with `kubectl get secret` can read them
- Never commit secrets to Git — they persist in history forever even if deleted
- Volume mounts over env vars: tighter permissions, no leak to child processes, can rotate without pod restart
- Enable Encryption at Rest in production; for real workloads, use an external secret manager (see Demo 10)

---

---

## Demo 10 — HashiCorp Vault + External Secrets Operator

> **12 min · minikube + Helm + Vault + ESO**  
> Show: store secrets in Vault, ESO syncs them to K8s Secrets, automatic rotation

### Setup

```bash
mkdir -p ~/devsecops-demo/demo-10 && cd ~/devsecops-demo/demo-10
```

### Step 1 — Install Vault (dev mode)

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set "server.dev.enabled=true" \
  --set "server.dev.devRootToken=root" \
  --set "injector.enabled=false"

kubectl wait --for=condition=Ready pod/vault-0 -n vault --timeout=120s
echo "Vault is running."
```

### Step 2 — Verify Vault status

```bash
kubectl exec -n vault vault-0 -- vault status
```

### Step 3 — Store secrets in Vault

```bash
kubectl exec -n vault vault-0 -- \
  vault secrets enable -path=secret kv-v2 2>/dev/null || \
  echo "KV engine already enabled"

kubectl exec -n vault vault-0 -- \
  vault kv put secret/myapp/database \
    username="appuser" \
    password="Vault\$ecure#2025" \
    host="postgres.internal.dangote.com" \
    port="5432"

kubectl exec -n vault vault-0 -- \
  vault kv put secret/myapp/api-keys \
    payment-gateway="pg_live_abc123xyz789" \
    sms-provider="sms_key_def456uvw"

echo "Secrets stored in Vault."
```

### Step 4 — Read secrets back to confirm

```bash
kubectl exec -n vault vault-0 -- vault kv get secret/myapp/database
```

### Step 5 — Create the app namespace and store the Vault token

```bash
kubectl create namespace app-demo

kubectl create secret generic vault-token \
  --from-literal=token=root \
  -n app-demo

echo "Vault token stored."
```

### Step 6 — Install External Secrets Operator

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

### Step 7 — Create a SecretStore (connection to Vault)

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
          name: vault-token
          key: token
EOF

kubectl apply -f secret-store.yaml
sleep 5
kubectl get secretstore vault-backend -n app-demo
```

**Expected:** `STATUS=Valid  READY=True`

### Step 8 — Create an ExternalSecret

```bash
cat > external-secret.yaml << 'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: app-demo
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
  - secretKey: DB_USERNAME
    remoteRef:
      key: myapp/database
      property: username
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

### Step 9 — Verify the Kubernetes Secret was auto-created

```bash
kubectl get externalsecret database-credentials -n app-demo
kubectl get secret db-credentials -n app-demo
kubectl get secret db-credentials -n app-demo -o jsonpath='{.data.DB_HOST}' | base64 -d
echo ""
```

**Expected:** `postgres.internal.dangote.com`

The K8s Secret was automatically created by ESO — applications consume it normally.

### Step 10 — Simulate secret rotation

```bash
kubectl exec -n vault vault-0 -- \
  vault kv put secret/myapp/database \
    username="appuser" \
    password="NewRotated\$ecure#2025" \
    host="postgres.internal.dangote.com" \
    port="5432"

echo "Waiting for ESO to sync (refresh interval: 1m)..."
sleep 65

kubectl get secret db-credentials -n app-demo \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
echo ""
```

**Expected:** `NewRotated$ecure#2025` — rotated automatically without any pod restart.

### Cleanup

```bash
kubectl delete namespace app-demo external-secrets vault
helm uninstall vault -n vault 2>/dev/null || true
helm uninstall external-secrets -n external-secrets 2>/dev/null || true
rm -rf ~/devsecops-demo/demo-10
```

**Key takeaways:**
- External secret managers keep secrets out of Kubernetes etcd entirely
- ESO bridges the gap — apps use normal K8s Secrets, source is Vault
- Secret rotation in Vault propagates automatically within the refresh interval — no manual updates
- Same pattern works with Azure Key Vault, AWS Secrets Manager, GCP Secret Manager — just change the SecretStore provider

---

---

## Demo 11 — Secret Scanning with Gitleaks

> **6 min · Gitleaks + git · no Kubernetes needed**  
> Show: detect secrets in history, pre-commit hook blocks commit

### Setup

```bash
mkdir -p ~/devsecops-demo/demo-11 && cd ~/devsecops-demo/demo-11
mkdir -p demo-repo && cd demo-repo
git init
git config user.email "demo@dangote.com"
git config user.name "Demo User"
```

### Step 1 — Commit files containing secrets

```bash
# Commit 1
cat > config.py << 'EOF'
DATABASE_HOST = "postgres.internal.example.com"
DATABASE_USER = "admin"
DATABASE_PASSWORD = "p@ssw0rd!SuperSecret2024"
EOF

git add config.py
git commit -m "Add database configuration"

# Commit 2: add API keys
cat > .env << 'EOF'
STRIPE_SECRET_KEY=sk_test_zQPyAkXOPdh2IEbhCDN0gZhMbJ83XYZABC12345
AWS_SECRET_ACCESS_KEY=P9Tq/ABcDefGHiJkL+MnOpQrStUvWxYzABcDeF12
GITHUB_TOKEN=ghp_ABcDefGHiJkLmNoPqRsTuVwXyZABcDeFgHiJ12
EOF

git add .env
git commit -m "Add API credentials config"

# Commit 3: "remove" the secrets — but history keeps them
cat > .env << 'EOF'
STRIPE_SECRET_KEY=
AWS_SECRET_ACCESS_KEY=
GITHUB_TOKEN=
EOF

git add .env
git commit -m "Fix: move secrets to environment variables"
```

### Step 2 — Scan the entire git history

```bash
gitleaks detect --source . --verbose 2>&1
```

**Expected:** `4 secrets detected` — including ones from "removed" commits.

> **Point out:** Removing a secret from HEAD does NOT remove it from git history.

### Step 3 — Prove the "removed" secrets are still there

```bash
git log --oneline

COMMIT=$(git log --oneline | grep "API credentials" | awk '{print $1}')
git show $COMMIT:.env | grep -E "KEY|TOKEN|SECRET"
```

**Expected:** All three secrets visible from the old commit.

### Step 4 — Install a pre-commit hook

```bash
mkdir -p .git/hooks

cat > .git/hooks/pre-commit << 'HOOK'
#!/bin/sh
echo "Running Gitleaks secret scan..."
gitleaks protect --staged --verbose
if [ $? -ne 0 ]; then
    echo ""
    echo "COMMIT BLOCKED: Gitleaks detected secrets in staged changes."
    echo "Remove the secrets, then commit again."
    exit 1
fi
echo "No secrets detected. Commit allowed."
HOOK

chmod +x .git/hooks/pre-commit
echo "Pre-commit hook installed."
```

### Step 5 — Try to commit a new secret (hook blocks it)

```bash
cat > new-secret.py << 'EOF'
STRIPE_SECRET_KEY = "sk_test_zQPyAkXOPdh2IEbhCDN0gZhMbJ83XYZABC12345"
EOF

git add new-secret.py
git commit -m "Add payment integration"
```

**Expected:**
```
Running Gitleaks secret scan...
Finding: stripe-access-token
COMMIT BLOCKED: Gitleaks detected secrets in staged changes.
```

### Cleanup

```bash
cd ~/devsecops-demo/demo-11
rm -rf demo-repo
rm -rf ~/devsecops-demo/demo-11
```

**Key takeaways:**
- Removing a secret from code does not remove it from git history — always rotate credentials that were ever committed
- Gitleaks scans full git history — secrets from months ago are still detectable
- Pre-commit hooks are the earliest prevention — stop secrets before they enter the repository
- When a secret is found: **1) Revoke immediately 2) Rotate 3) Then clean history with `git filter-repo`**

---

---

## Demo 12 — Runtime Threat Detection with Falco

> **10 min · separate minikube cluster + Helm + Falco**  
> Platform: **Linux / Windows WSL2 only** — macOS Docker Desktop does not support eBPF in nested containers

### Start a dedicated cluster

```bash
minikube start \
  --profile=falco-demo \
  --cpus=2 \
  --memory=4096 \
  --driver=docker

kubectl config use-context falco-demo
mkdir -p ~/devsecops-demo/demo-12 && cd ~/devsecops-demo/demo-12
```

### Step 1 — Install Falco with modern eBPF driver

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=modern_ebpf \
  --set tty=true

echo "Waiting for Falco to initialize..."
kubectl wait --for=condition=Ready pods --all -n falco --timeout=180s
echo "Falco is running."
```

### Step 2 — Verify Falco is monitoring

```bash
kubectl get pods -n falco
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=10 | \
  grep -E "version|Starting|Enabled|Opening|Loading"
```

**Expected:** `Opening 'syscall' source with modern BPF probe.`

### Step 3 — Deploy a test workload

```bash
kubectl create namespace falco-test

cat > test-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-workload
  namespace: falco-test
spec:
  containers:
  - name: app
    image: ubuntu:22.04
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
EOF

kubectl apply -f test-pod.yaml
kubectl wait --for=condition=Ready pod/test-workload -n falco-test --timeout=60s
echo "Test pod ready."
```

### Step 4 — Trigger: shell spawned in container

```bash
echo "=== Triggering: Shell spawned in container ==="
kubectl exec -n falco-test test-workload -- bash -c "echo 'Attacker was here'"

sleep 3
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=20 | \
  grep -i "shell\|Terminal" | head -5
```

**Expected Falco alert:**
```
Notice A shell was spawned in a container ... rule=Terminal shell in container
```

### Step 5 — Trigger: reading sensitive files

```bash
echo "=== Triggering: Read sensitive file ==="
kubectl exec -n falco-test test-workload -- bash -c "cat /etc/passwd"

sleep 3
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=30 | \
  grep -i "sensitive\|passwd" | head -5
```

### Step 6 — View all alerts with formatted output

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=100 | \
  grep '"rule"' | \
  python3 -c "
import sys, json
for line in sys.stdin:
    try:
        entry = json.loads(line.strip())
        priority = entry.get('priority', '')
        rule = entry.get('rule', '')
        prefix = {'Warning': '🟡', 'Notice': '🔵', 'Critical': '🔴'}.get(priority, '▪')
        print(f'{prefix} [{priority}] {rule}')
    except:
        pass
" 2>/dev/null | sort | uniq -c | sort -rn | head -10
```

### Step 7 — Custom rule example (reference only)

```bash
cat > custom-rule-example.yaml << 'EOF'
- macro: is_production_namespace
  condition: k8s.ns.name in (production, prod)

- list: approved_base_images
  items: ["nginx:1.25-alpine", "python:3.11-alpine"]

- rule: Unapproved image in production
  desc: Container started with an image not in the approved list
  condition: >
    container.started and
    is_production_namespace and
    not container.image.repository in (approved_base_images)
  output: >
    Unapproved image launched (image=%container.image.repository:%container.image.tag
    pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: WARNING
  tags: [container, supply-chain]
EOF

echo "Custom rule example — deploy with:"
echo "  helm upgrade falco falcosecurity/falco --set-file customRules.rules.yaml=custom-rule-example.yaml"
```

### Cleanup

```bash
rm -rf ~/devsecops-demo/demo-12
kubectl delete namespace falco-test
minikube stop --profile=falco-demo
minikube delete --profile=falco-demo
kubectl config use-context minikube
```

**Key takeaways:**
- Static scanning finds vulnerabilities — Falco detects attacks at runtime. Both are needed
- Falco uses eBPF to monitor syscalls — sees everything without modifying the application
- Common indicators: shell spawned in container, writes to `/etc`, reading `/proc/*/environ`
- Route alerts to Slack/PagerDuty via `falcosidekick` — raw logs on the node are not actionable at scale

---

---

## Quick Reference — Cleanup All

If anything gets stuck, clean up everything at once:

```bash
# Stop and remove all minikube profiles
minikube delete --all

# Remove demo files
rm -rf ~/devsecops-demo

# Remove demo Docker images
docker rmi demo-app:insecure demo-app:hardened 2>/dev/null || true
```

Restart fresh:
```bash
minikube start --cpus=2 --memory=4096 --driver=docker
kubectl get nodes
```
