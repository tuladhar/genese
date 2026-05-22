# Demo 04: Namespace & Node Isolation

> **Duration:** ~8 minutes  
> **Section:** 3.2 — Workload Isolation, Namespace Isolation, Taints & Tolerations  
> **Tools:** minikube, kubectl

---

## Learning Objectives

- Create namespaces representing different environments (dev, staging, prod)
- Apply different Pod Security Standard levels per namespace
- Demonstrate workload isolation using namespace-scoped RBAC
- Configure taints and tolerations to isolate sensitive workloads to specific nodes

---

## Prerequisites

- Minikube running: `minikube status`
- If not started: `minikube start --cpus=2 --memory=4096 --driver=docker`

---

## Setup

```bash
mkdir -p ~/devsecops-demo/demo-04 && cd ~/devsecops-demo/demo-04
```

---

## Part 1: Namespace Isolation with Pod Security Standards

### Step 1 — Create environment namespaces

```bash
kubectl create namespace dev 2>/dev/null || kubectl label namespace dev app=demo --overwrite
kubectl create namespace staging 2>/dev/null || kubectl label namespace staging app=demo --overwrite
kubectl create namespace production 2>/dev/null || kubectl label namespace production app=demo --overwrite
```

### Step 2 — Apply different security levels per namespace

```bash
# dev: warn only — developers see warnings but pods still run
kubectl label namespace dev \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=latest

# staging: enforce baseline — blocks known privilege escalations
kubectl label namespace staging \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest

# production: enforce restricted — strictest standard
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest
```

### Step 3 — Verify labels are applied

```bash
kubectl get namespace dev staging production --show-labels | grep pod-security
```

### Step 4 — Test isolation across namespaces

Deploy the same pod to each namespace and observe the behavior:

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
      runAsUser: 0            # root user — blocked by restricted/baseline
      privileged: false
EOF
```

```bash
# dev: allowed (warn only)
kubectl apply -f test-pod.yaml -n dev
echo "dev result: $?"

# staging: blocked (baseline enforced — runAsUser: 0 is not a baseline violation,
# but privileged would be. Let's show a privileged pod blocked)
kubectl apply -f test-pod.yaml -n staging
echo "staging result: $?"

# production: blocked (restricted enforced — root user not allowed)
kubectl apply -f test-pod.yaml -n production
echo "production result: $?"
```

**Expected output:**
```
# dev — pod created with a warning
Warning: would violate PodSecurity "baseline:latest": ...
pod/test-pod created

# staging — pod created (runAsUser:0 is baseline-compliant)
pod/test-pod created

# production — BLOCKED
Error from server (Forbidden): pods "test-pod" is forbidden: violates PodSecurity "restricted:latest":
  runAsNonRoot != true ...
```

---

## Part 2: Namespace-Scoped RBAC Isolation

Namespaces become truly isolated when combined with RBAC — a developer in `dev` cannot see resources in `production`.

### Step 5 — Create a developer service account

```bash
kubectl create serviceaccount dev-user -n dev
```

### Step 6 — Grant the dev-user access only to the dev namespace

```bash
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

### Step 7 — Verify namespace isolation

```bash
# Can dev-user list pods in dev? YES
kubectl auth can-i list pods -n dev \
  --as=system:serviceaccount:dev:dev-user

# Can dev-user list pods in production? NO
kubectl auth can-i list pods -n production \
  --as=system:serviceaccount:dev:dev-user

# Can dev-user list pods cluster-wide? NO
kubectl auth can-i list pods --all-namespaces \
  --as=system:serviceaccount:dev:dev-user
```

**Expected output:**
```
yes
no
no
```

---

## Part 3: Node Isolation with Taints and Tolerations

In a real cluster, you might want sensitive production workloads on dedicated, hardened nodes.

### Step 8 — View existing nodes

```bash
kubectl get nodes --show-labels
```

With minikube, there is typically one node. In a real cluster, you'd have multiple nodes. We'll demonstrate the mechanism.

### Step 9 — Taint the minikube node as a "secure" node

```bash
# Add a taint: only workloads that tolerate this taint can run here
kubectl taint nodes minikube workload-type=secure:NoSchedule
```

### Step 10 — Try deploying without a toleration

```bash
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

**Expected output:**
```
NAME                 STATUS    REASON
pod-no-toleration    Pending   0/1 nodes are available: 1 node(s) had untolerated taint {workload-type: secure}
```

The pod stays in `Pending` — it cannot schedule on the tainted node.

### Step 11 — Deploy with the correct toleration

```bash
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

**Expected output:**
```
NAME                   STATUS    NODE
pod-with-toleration    Running   minikube
```

---

## Isolation Architecture Summary

```
┌─────────────────────────────────────────┐
│              Kubernetes Cluster          │
│                                          │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │   dev    │  │ staging  │  │  prod  │ │
│  │ (warn)   │  │(baseline)│  │(restr.)│ │
│  │          │  │          │  │        │ │
│  │dev-user  │  │          │  │        │ │
│  │(RBAC:ns) │  │          │  │        │ │
│  └──────────┘  └──────────┘  └────────┘ │
│                                          │
│  Node: [taint: workload-type=secure]     │
│  Only tolerating pods can schedule here  │
└─────────────────────────────────────────┘
```

---

## Cleanup

```bash
# Remove taint
kubectl taint nodes minikube workload-type=secure:NoSchedule-

# Remove resources
kubectl delete pod pod-no-toleration pod-with-toleration -n dev --ignore-not-found
kubectl delete pod test-pod -n dev --ignore-not-found
kubectl delete pod test-pod -n staging --ignore-not-found
kubectl delete namespace dev staging production
rm -rf ~/devsecops-demo/demo-04
```

---

## Key Takeaways

1. **Namespaces are not security boundaries by themselves** — they need PSA labels and RBAC to enforce isolation.
2. Apply the **strictest Pod Security Standard** to production namespaces; use looser settings in dev for developer experience.
3. **RBAC + Namespaces** together ensure a compromised service account in `dev` cannot access `production` resources.
4. **Taints and tolerations** provide node-level isolation for sensitive workloads (e.g., databases, payment services).
5. In a real multi-node cluster, combine taints with **node affinity** for deterministic placement of secure workloads.
