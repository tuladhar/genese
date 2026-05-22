# Demo 07: RBAC in Action

> **Duration:** ~8 minutes  
> **Section:** 3.4 — Kubernetes RBAC  
> **Tools:** minikube, kubectl

---

## Learning Objectives

- Understand Kubernetes RBAC objects: Role, ClusterRole, RoleBinding, ClusterRoleBinding
- Apply the principle of least privilege to service accounts
- Test and audit permissions using `kubectl auth can-i`
- Demonstrate how excessive RBAC permissions lead to privilege escalation

---

## Prerequisites

- Default minikube cluster running: `minikube status`
- If not started: `minikube start --cpus=2 --memory=4096 --driver=docker`

---

## Setup

```bash
mkdir -p ~/devsecops-demo/demo-07 && cd ~/devsecops-demo/demo-07

kubectl create namespace rbac-demo
```

---

## Part 1: The Over-Privileged Service Account

### Step 1 — Create a service account with cluster-admin (dangerous)

This is a common mistake: giving a service account admin rights for convenience.

```bash
kubectl create serviceaccount over-privileged-sa -n rbac-demo

# Binding to cluster-admin gives FULL control over the entire cluster
kubectl create clusterrolebinding over-privileged-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=rbac-demo:over-privileged-sa
```

### Step 2 — Test what this service account can do

```bash
SA="system:serviceaccount:rbac-demo:over-privileged-sa"

echo "Can it list secrets in kube-system (should be NO in prod)?"
kubectl auth can-i list secrets -n kube-system --as=$SA

echo "Can it delete nodes?"
kubectl auth can-i delete nodes --as=$SA

echo "Can it create ClusterRoleBindings (privilege escalation risk)?"
kubectl auth can-i create clusterrolebindings --as=$SA

echo "Can it access everything?"
kubectl auth can-i '*' '*' --as=$SA
```

**Expected output:**
```
yes
yes
yes
yes
```

A compromised pod using this service account has **full cluster control**. This is one of the most critical misconfigurations in real clusters.

---

## Part 2: Least-Privilege Service Account

### Step 3 — Create a properly scoped service account

**Scenario:** A backend API service needs to read ConfigMaps and Secrets in its own namespace only. Nothing else.

```bash
# Create the service account
kubectl create serviceaccount backend-api-sa -n rbac-demo
```

### Step 4 — Define a minimal Role

```bash
cat > backend-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backend-api-role
  namespace: rbac-demo
rules:
  # Can read ConfigMaps (for application configuration)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
  # Can read Secrets (for database credentials)
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  # Can read its own pod info (for health checks, etc.)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF

kubectl apply -f backend-role.yaml
```

### Step 5 — Bind the role to the service account

```bash
cat > backend-rolebinding.yaml << 'EOF'
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

kubectl apply -f backend-rolebinding.yaml
```

### Step 6 — Verify least-privilege permissions

```bash
SA_BACKEND="system:serviceaccount:rbac-demo:backend-api-sa"

echo "=== Allowed operations ==="
echo -n "  list configmaps in rbac-demo:    "
kubectl auth can-i list configmaps -n rbac-demo --as=$SA_BACKEND

echo -n "  get secrets in rbac-demo:        "
kubectl auth can-i get secrets -n rbac-demo --as=$SA_BACKEND

echo ""
echo "=== Blocked operations ==="
echo -n "  delete pods in rbac-demo:        "
kubectl auth can-i delete pods -n rbac-demo --as=$SA_BACKEND

echo -n "  list secrets in kube-system:     "
kubectl auth can-i list secrets -n kube-system --as=$SA_BACKEND

echo -n "  list pods cluster-wide:          "
kubectl auth can-i list pods --all-namespaces --as=$SA_BACKEND

echo -n "  create ClusterRoleBindings:      "
kubectl auth can-i create clusterrolebindings --as=$SA_BACKEND
```

**Expected output:**
```
=== Allowed operations ===
  list configmaps in rbac-demo:    yes
  get secrets in rbac-demo:        yes

=== Blocked operations ===
  delete pods in rbac-demo:        no
  list secrets in kube-system:     no
  list pods cluster-wide:          no
  create ClusterRoleBindings:      no
```

---

## Part 3: Role vs ClusterRole

### Step 7 — Understand scope

```bash
cat > readonly-clusterrole.yaml << 'EOF'
# ClusterRole: permissions apply across all namespaces when bound with ClusterRoleBinding
# But can also be scoped to one namespace with a RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
EOF

kubectl apply -f readonly-clusterrole.yaml

# Bind it as a RoleBinding (scoped to rbac-demo namespace only)
kubectl create rolebinding pod-reader-ns-scoped \
  --clusterrole=pod-reader \
  --serviceaccount=rbac-demo:backend-api-sa \
  --namespace=rbac-demo

# Verify: can read pods in rbac-demo
kubectl auth can-i list pods -n rbac-demo \
  --as=system:serviceaccount:rbac-demo:backend-api-sa

# Verify: cannot read pods in kube-system
kubectl auth can-i list pods -n kube-system \
  --as=system:serviceaccount:rbac-demo:backend-api-sa
```

**Expected output:**
```
yes
no
```

A `ClusterRole` bound with a `RoleBinding` is scoped to that namespace — the ClusterRole just reuses the permission definition.

---

## Part 4: Audit RBAC — Who Can Do What

### Step 8 — List all ClusterRoleBindings that grant cluster-admin

```bash
echo "=== Accounts with cluster-admin ==="
kubectl get clusterrolebindings -o json | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
for item in data['items']:
    if item.get('roleRef', {}).get('name') == 'cluster-admin':
        subjects = item.get('subjects', [])
        for s in subjects:
            print(f\"  {s.get('kind')}/{s.get('namespace', 'cluster')}/{s.get('name')} via {item['metadata']['name']}\")
"
```

**Expected output (includes our overprivileged account):**
```
  ServiceAccount/rbac-demo/over-privileged-sa via over-privileged-binding
  User/cluster/admin via cluster-admin (minikube default)
```

### Step 9 — Check all permissions for a service account

```bash
kubectl auth can-i --list -n rbac-demo \
  --as=system:serviceaccount:rbac-demo:backend-api-sa
```

This lists every operation the service account is authorized to perform.

---

## RBAC Object Reference

```
┌─────────────────────────────────────────────────────────────┐
│                    RBAC Relationships                        │
│                                                              │
│  Role (namespace-scoped)          ClusterRole (global)       │
│  └── defines: what verbs          └── defines: what verbs    │
│      on what resources                on what resources       │
│                                                              │
│  RoleBinding (namespace-scoped)   ClusterRoleBinding         │
│  ├── subject: User/Group/SA       ├── subject: User/Group/SA │
│  └── roleRef: Role or ClusterRole └── roleRef: ClusterRole   │
└─────────────────────────────────────────────────────────────┘
```

---

## Cleanup

```bash
kubectl delete namespace rbac-demo
kubectl delete clusterrolebinding over-privileged-binding
kubectl delete clusterrole pod-reader
rm -rf ~/devsecops-demo/demo-07
```

---

## Key Takeaways

1. **Never bind `cluster-admin`** to application service accounts — this grants full cluster control.
2. **Use Roles (not ClusterRoles)** when permissions are needed only in one namespace.
3. **`kubectl auth can-i --list`** is your audit tool — run it on every service account before deploying.
4. Default service accounts have no permissions in modern Kubernetes — good. Don't add `automountServiceAccountToken: true` unless needed.
5. The principle of least privilege means: read-only where possible, namespace-scoped where possible, no wildcard verbs (`*`).
