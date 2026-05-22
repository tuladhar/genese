# Demo 03: Pod Hardening & Pod Security Admission

> **Duration:** ~10 minutes  
> **Section:** 3.2 — Pod Security  
> **Tools:** minikube, kubectl

---

## Learning Objectives

- Deploy an intentionally insecure pod and identify its weaknesses
- Apply Pod Security Standards (PSS) via Pod Security Admission (PSA)
- Harden a pod manifest with security contexts
- Observe how PSA blocks non-compliant workloads

---

## Prerequisites

- Minikube running: `minikube status`
- If not started: `minikube start --cpus=2 --memory=4096 --driver=docker`

---

## Setup

```bash
mkdir -p ~/devsecops-demo/demo-03 && cd ~/devsecops-demo/demo-03
```

---

## Part 1: The Insecure Pod

### Step 1 — Deploy an insecure pod

This pod has all the worst-practice security settings — common in real workloads.

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
      privileged: true          # Full host access — critical risk
      runAsUser: 0              # Running as root
      allowPrivilegeEscalation: true
    resources: {}               # No CPU/memory limits
EOF

kubectl apply -f pod-insecure.yaml
kubectl wait --for=condition=Ready pod/insecure-pod --timeout=60s
```

### Step 2 — Examine the running process

```bash
# What user is the process running as?
kubectl exec insecure-pod -- id

# Can it write to the filesystem?
kubectl exec insecure-pod -- sh -c "echo 'pwned' > /tmp/test && cat /tmp/test"
```

**Expected output:**
```
uid=0(root) gid=0(root) groups=0(root),...
pwned
```

A compromised process inside this container runs as **root** and has **full filesystem write access**.

### Step 3 — Check if privilege escalation is possible

```bash
kubectl exec insecure-pod -- cat /proc/1/status | grep -E "CapEff|CapPrm"
```

The capabilities bitmask will show all capabilities enabled — a complete privilege set.

---

## Part 2: Applying Pod Security Standards

Kubernetes provides three built-in security profiles enforced via namespace labels:

| Profile | Description |
|---------|-------------|
| `privileged` | Unrestricted — allows everything |
| `baseline` | Prevents known privilege escalations |
| `restricted` | Hardened — follows current best practices |

### Step 4 — Create a restricted namespace

```bash
kubectl create namespace secure-ns

# Label the namespace to enforce the "restricted" Pod Security Standard
kubectl label namespace secure-ns \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest
```

### Step 5 — Try to deploy the insecure pod in the secure namespace

```bash
kubectl apply -f pod-insecure.yaml -n secure-ns
```

**Expected output:**
```
Error from server (Forbidden): error when creating "pod-insecure.yaml": pods "insecure-pod" is forbidden: violates PodSecurity "restricted:latest":
  allowPrivilegeEscalation != false (container "app" must set securityContext.allowPrivilegeEscalation=false)
  privileged (container "app" must not set securityContext.privileged=true)
  runAsNonRoot != true (pod or container "app" must set securityContext.runAsNonRoot=true)
  ...
```

Pod Security Admission **blocked the deployment** — the pod never started.

---

## Part 3: The Hardened Pod

### Step 6 — Deploy a compliant, hardened pod

```bash
cat > pod-hardened.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
  namespace: secure-ns
spec:
  securityContext:
    runAsNonRoot: true          # Pod-level: all containers must run as non-root
    runAsUser: 1000             # Explicit non-root UID
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault      # Use the runtime's default seccomp profile
  containers:
  - name: app
    image: nginx:1.25-alpine
    securityContext:
      allowPrivilegeEscalation: false   # Cannot gain more privileges
      readOnlyRootFilesystem: true      # Filesystem is read-only
      capabilities:
        drop:
          - ALL                         # Drop all Linux capabilities
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
EOF

kubectl apply -f pod-hardened.yaml
kubectl wait --for=condition=Ready pod/hardened-pod -n secure-ns --timeout=60s
```

### Step 7 — Verify the hardened pod

```bash
# User context
kubectl exec -n secure-ns hardened-pod -- id
```

**Expected output:**
```
uid=1000 gid=1000 groups=1000
```

```bash
# Read-only filesystem — can't write outside allowed volumes
kubectl exec -n secure-ns hardened-pod -- sh -c "echo test > /etc/test.txt" 2>&1
```

**Expected output:**
```
sh: can't create /etc/test.txt: Read-only file system
```

```bash
# Writable /tmp is available (mounted as emptyDir)
kubectl exec -n secure-ns hardened-pod -- sh -c "echo 'tmp works' > /tmp/test && cat /tmp/test"
```

**Expected output:**
```
tmp works
```

---

## Security Context Comparison

| Setting | Insecure Pod | Hardened Pod |
|---------|-------------|--------------|
| `runAsNonRoot` | No (UID 0) | Yes (UID 1000) |
| `privileged` | true | false |
| `allowPrivilegeEscalation` | true | false |
| `readOnlyRootFilesystem` | No | Yes |
| `capabilities` | All | None (drop: ALL) |
| `seccompProfile` | None | RuntimeDefault |
| Resource limits | None | CPU + Memory limits set |

---

## Cleanup

```bash
kubectl delete pod insecure-pod
kubectl delete pod hardened-pod -n secure-ns
kubectl delete namespace secure-ns
rm -rf ~/devsecops-demo/demo-03
```

---

## Key Takeaways

1. **Pod Security Admission (PSA)** is built into Kubernetes — no extra tools needed to enforce security standards.
2. **Namespace labels** control the enforcement level — apply `restricted` to production namespaces.
3. `runAsNonRoot`, `readOnlyRootFilesystem`, and `drop: ALL` capabilities are the three most impactful hardening settings.
4. **Resource limits** prevent a compromised pod from consuming all cluster resources (denial of service).
5. PSA blocks non-compliant pods at admission time — before any code runs.
