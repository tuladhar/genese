# Demo 05: Enforcing Kubernetes Network Policy

> **Duration:** ~10 minutes  
> **Section:** 3.3 — Kubernetes Network Security  
> **Tools:** minikube (Calico CNI), kubectl

---

## Learning Objectives

- Understand that Kubernetes has no network isolation by default
- Create a default-deny NetworkPolicy to block all traffic
- Allow only specific pod-to-pod communication paths
- Test network connectivity before and after applying policies

---

## Prerequisites

> **This demo requires a separate minikube cluster with Calico CNI.**  
> The default minikube setup does not enforce NetworkPolicy rules — a CNI plugin that supports policy enforcement is required.

### Start a Calico-enabled cluster

```bash
# Start a named profile to avoid disrupting your main cluster
minikube start \
  --profile=netpolicy \
  --cpus=2 \
  --memory=4096 \
  --driver=docker \
  --cni=calico

# Set this profile as active for this demo
export MINIKUBE_PROFILE=netpolicy
kubectl config use-context netpolicy

# Wait for Calico to be ready (may take 2-3 minutes)
kubectl wait --for=condition=Ready pods --all -n kube-system --timeout=180s
echo "Cluster ready."
```

---

## Setup

```bash
mkdir -p ~/devsecops-demo/demo-05 && cd ~/devsecops-demo/demo-05

# Create a demo namespace
kubectl create namespace netdemo
```

---

## Part 1: Default Behavior — All Traffic Allowed

### Step 1 — Deploy three application tiers

```bash
cat > apps.yaml << 'EOF'
# Frontend
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: netdemo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
      tier: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
---
# Backend API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: netdemo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      tier: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
      - name: backend
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
---
# Database
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: netdemo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: database
  template:
    metadata:
      labels:
        app: database
        tier: database
    spec:
      containers:
      - name: database
        image: nginx:1.25-alpine  # simulating a database
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: netdemo
spec:
  selector:
    app: frontend
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: netdemo
spec:
  selector:
    app: backend
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: database-svc
  namespace: netdemo
spec:
  selector:
    app: database
  ports:
  - port: 80
EOF

kubectl apply -f apps.yaml
kubectl wait --for=condition=Available deployment/frontend deployment/backend deployment/database \
  -n netdemo --timeout=90s
echo "All pods ready."
```

### Step 2 — Test connectivity (all should succeed)

```bash
FRONTEND_POD=$(kubectl get pod -n netdemo -l app=frontend -o jsonpath='{.items[0].metadata.name}')

echo "=== Frontend → Backend (should succeed) ==="
kubectl exec -n netdemo $FRONTEND_POD -- wget -qO- --timeout=3 http://backend-svc && echo "CONNECTED"

echo ""
echo "=== Frontend → Database (should succeed — but this is a security problem!) ==="
kubectl exec -n netdemo $FRONTEND_POD -- wget -qO- --timeout=3 http://database-svc && echo "CONNECTED"
```

**Expected output:**
```
=== Frontend → Backend (should succeed) ===
CONNECTED

=== Frontend → Database (should succeed — but this is a security problem!) ===
CONNECTED
```

Without NetworkPolicy, the frontend can directly reach the database. This is a classic lateral movement risk.

---

## Part 2: Enforce Default Deny

### Step 3 — Apply default-deny for ingress and egress

```bash
cat > default-deny.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: netdemo
spec:
  podSelector: {}       # Applies to ALL pods in the namespace
  policyTypes:
  - Ingress
  - Egress
EOF

kubectl apply -f default-deny.yaml
echo "Default deny applied."
```

### Step 4 — Test connectivity again (all should fail)

```bash
echo "=== Frontend → Backend (should FAIL) ==="
kubectl exec -n netdemo $FRONTEND_POD -- wget -qO- --timeout=3 http://backend-svc 2>&1 && echo "CONNECTED" || echo "BLOCKED"

echo ""
echo "=== Frontend → Database (should FAIL) ==="
kubectl exec -n netdemo $FRONTEND_POD -- wget -qO- --timeout=3 http://database-svc 2>&1 && echo "CONNECTED" || echo "BLOCKED"
```

**Expected output:**
```
=== Frontend → Backend (should FAIL) ===
BLOCKED

=== Frontend → Database (should FAIL) ===
BLOCKED
```

All traffic is now blocked. Now we selectively re-open only what's needed.

---

## Part 3: Selective Allow Policies

### Step 5 — Allow frontend to communicate with backend only

```bash
cat > allow-frontend-backend.yaml << 'EOF'
# Allow frontend pods to make outbound connections to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: netdemo
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 80
  # DNS resolution — required for service name lookup
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# Allow backend to receive connections from frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-ingress-from-frontend
  namespace: netdemo
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
EOF

kubectl apply -f allow-frontend-backend.yaml
```

### Step 6 — Allow backend to communicate with database only

```bash
cat > allow-backend-database.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: netdemo
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 80
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-ingress-from-backend
  namespace: netdemo
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 80
EOF

kubectl apply -f allow-backend-database.yaml
```

### Step 7 — Test the enforced traffic matrix

```bash
BACKEND_POD=$(kubectl get pod -n netdemo -l app=backend -o jsonpath='{.items[0].metadata.name}')

echo "=== Frontend → Backend (should SUCCEED) ==="
kubectl exec -n netdemo $FRONTEND_POD -- wget -qO- --timeout=3 http://backend-svc && echo "CONNECTED" || echo "BLOCKED"

echo ""
echo "=== Frontend → Database (should FAIL — no direct path) ==="
kubectl exec -n netdemo $FRONTEND_POD -- wget -qO- --timeout=3 http://database-svc 2>&1 && echo "CONNECTED" || echo "BLOCKED"

echo ""
echo "=== Backend → Database (should SUCCEED) ==="
kubectl exec -n netdemo $BACKEND_POD -- wget -qO- --timeout=3 http://database-svc && echo "CONNECTED" || echo "BLOCKED"
```

**Expected output:**
```
=== Frontend → Backend (should SUCCEED) ===
CONNECTED

=== Frontend → Database (should FAIL — no direct path) ===
BLOCKED

=== Backend → Database (should SUCCEED) ===
CONNECTED
```

---

## Enforced Traffic Matrix

```
Internet
   │
   ▼
[frontend] ──────► [backend] ──────► [database]
               ✗                 ✗
         (frontend cannot      (backend cannot
          reach database)       skip to internet)
```

| Source | Destination | Result |
|--------|------------|--------|
| frontend | backend | ✓ Allowed |
| frontend | database | ✗ Blocked |
| backend | database | ✓ Allowed |
| backend | internet | ✗ Blocked |
| database | anywhere | ✗ Blocked |

---

## Cleanup

```bash
kubectl delete namespace netdemo
rm -rf ~/devsecops-demo/demo-05

# Stop the netpolicy cluster when done with this demo
minikube stop --profile=netpolicy

# To resume other demos, switch back to your default cluster
kubectl config use-context minikube
```

---

## Key Takeaways

1. **Kubernetes allows all pod-to-pod traffic by default** — a compromised frontend can reach your database.
2. **NetworkPolicy requires a CNI plugin** that supports enforcement — Calico, Cilium, or Weave. The default minikube CNI does not enforce policies.
3. Start with **default-deny-all** then add selective allow rules — always easier than trying to identify what to block.
4. **DNS egress (UDP 53 to kube-system)** must be explicitly allowed when using default-deny-egress, or service name lookups will fail.
5. NetworkPolicy is **namespace-scoped** — each namespace needs its own policies.
