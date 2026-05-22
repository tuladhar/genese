# Demo 08: Enabling Kubernetes Audit Logging

> **Duration:** ~8 minutes  
> **Section:** 3.4 — Kubernetes API Security, Audit Logging  
> **Tools:** minikube, kubectl

---

## Learning Objectives

- Understand what Kubernetes audit logging captures and why it matters
- Configure an audit policy to record security-sensitive API operations
- Start minikube with audit logging enabled
- Read and interpret audit log entries for real operations

---

## Prerequisites

- minikube installed: `minikube version`

> **Note:** This demo requires restarting minikube with a custom configuration. If you have other demos running, they will be unaffected if you use a named profile.

---

## Setup

```bash
mkdir -p ~/devsecops-demo/demo-08 && cd ~/devsecops-demo/demo-08
```

---

## Background

Audit logging answers: **"Who did what, to which resource, and when?"**

Kubernetes audit levels:

| Level | What is recorded |
|-------|-----------------|
| `None` | Nothing |
| `Metadata` | Request metadata (user, resource, verb) — no request/response body |
| `Request` | Metadata + request body |
| `RequestResponse` | Metadata + request body + response body |

Higher levels provide more detail but also more log volume.

---

## Part 1: Create the Audit Policy

### Step 1 — Write the audit policy

```bash
cat > audit-policy.yaml << 'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all Secret operations with full request/response (most sensitive)
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["secrets"]

  # Log pod creation/deletion with request body
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    resources:
    - group: ""
      resources: ["pods"]

  # Log RBAC changes — critical for detecting privilege escalation
  - level: RequestResponse
    resources:
    - group: "rbac.authorization.k8s.io"
      resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]

  # Log all authentication and authorization failures at metadata level
  - level: Metadata
    omitStages:
    - RequestReceived
    users:
    - "system:anonymous"

  # Skip noisy health check endpoints
  - level: None
    nonResourceURLs:
    - "/healthz"
    - "/readyz"
    - "/livez"
    - "/metrics"

  # Skip kube-proxy watch traffic (too noisy)
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: ""
      resources: ["endpoints", "services", "services/status"]

  # Skip kubelets getting their own node/pod data
  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get"]
    resources:
    - group: ""
      resources: ["nodes", "nodes/status"]

  # Default: capture metadata for everything else
  - level: Metadata
    omitStages:
    - RequestReceived
EOF
```

---

## Part 2: Configure Minikube with Audit Logging

### Step 2 — Copy the audit policy into minikube

Start or ensure minikube is running, copy the policy into the VM, then restart with audit flags.

> **Note for Docker driver (Windows/Linux default):** The API server runs inside the minikube container. We use `--audit-log-path=-` to write audit events to the API server's stdout, which is captured by Kubernetes and readable via `kubectl logs`. This avoids needing an extra volume mount.

```bash
# Start minikube if not running (default profile)
minikube start --cpus=2 --memory=4096 --driver=docker

# Copy the audit policy into the minikube VM filesystem
minikube ssh "sudo mkdir -p /etc/ssl/certs"
cat audit-policy.yaml | minikube ssh "sudo tee /etc/ssl/certs/audit-policy.yaml > /dev/null"

# Verify the file is in place
minikube ssh "sudo head -5 /etc/ssl/certs/audit-policy.yaml"
```

### Step 3 — Restart minikube with audit logging enabled

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

> `--audit-log-path=-` writes events to the API server's stderr stream. This is the simplest approach for minikube with the Docker driver, where the `/var/log/` path is not directly accessible from the API server container. In production clusters, use a file path with rotation flags (`--audit-log-maxage`, `--audit-log-maxbackup`, `--audit-log-maxsize`).

---

## Part 3: Generate and Observe Audit Events

### Step 4 — Create a namespace and a secret (these will be logged)

```bash
kubectl create namespace audit-test
kubectl create secret generic db-password \
  --from-literal=password=SuperSecret123 \
  -n audit-test
```

### Step 5 — Read the secret (this access will be logged)

```bash
kubectl get secret db-password -n audit-test -o yaml
```

### Step 6 — Create an RBAC binding (high-value audit event)

```bash
kubectl create serviceaccount audit-sa -n audit-test
kubectl create rolebinding audit-test-binding \
  --clusterrole=view \
  --serviceaccount=audit-test:audit-sa \
  -n audit-test
```

### Step 7 — View the audit log

With `--audit-log-path=-`, audit events are written to the API server's stderr, readable via `kubectl logs`:

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

        # Filter to show only interesting operations
        if resource in ('secrets', 'rolebindings', 'clusterroles') or 'admin' in user:
            print(f'[{level}] {user} | {verb} {resource}/{name} (ns:{namespace}) -> HTTP {status}')
    except:
        pass
" 2>/dev/null | head -20
```

**Sample output:**
```
[RequestResponse] kubernetes-admin | create secrets/db-password (ns:audit-test) → HTTP 201
[RequestResponse] kubernetes-admin | get secrets/db-password (ns:audit-test) → HTTP 200
[RequestResponse] kubernetes-admin | create rolebindings/audit-test-binding (ns:audit-test) → HTTP 201
[Metadata] kubernetes-admin | create serviceaccounts/audit-sa (ns:audit-test) → HTTP 201
```

### Step 8 — Look at a full audit entry for a secret access

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
" 2>/dev/null | head -40
```

**Expected output (truncated):**
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "RequestResponse",
  "auditID": "abc123...",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/audit-test/secrets/db-password",
  "verb": "get",
  "user": {
    "username": "kubernetes-admin",
    "groups": ["system:masters", "system:authenticated"]
  },
  "objectRef": {
    "resource": "secrets",
    "namespace": "audit-test",
    "name": "db-password"
  },
  "responseStatus": { "code": 200 },
  "requestReceivedTimestamp": "2025-01-01T10:00:00Z"
}
```

---

## Audit Policy Design Guidelines

| Scenario | Recommended Level |
|----------|------------------|
| Secret access (read/write) | `RequestResponse` |
| Pod create/delete | `Request` |
| RBAC changes | `RequestResponse` |
| Node operations | `Metadata` |
| Health check endpoints | `None` |
| kube-proxy watch traffic | `None` |

---

## Cleanup

```bash
kubectl delete namespace audit-test
rm -rf ~/devsecops-demo/demo-08

# Restart minikube without audit flags for subsequent demos
minikube stop
minikube start --cpus=2 --memory=4096 --driver=docker

echo "Cluster restored to default config."
```

---

## Key Takeaways

1. **Audit logs are your forensic trail** — without them, you cannot determine what happened during a security incident.
2. **Don't log everything at `RequestResponse`** — log volume explodes. Be strategic: secrets and RBAC at `RequestResponse`, everything else at `Metadata`.
3. `None` for health checks and kube-proxy **reduces noise** so real events are visible.
4. In production, **ship audit logs to a SIEM** (Splunk, ELK, Datadog) — not just a file on the node.
5. Audit logging catches **insider threats** (admins accessing secrets directly) as well as external attackers.
