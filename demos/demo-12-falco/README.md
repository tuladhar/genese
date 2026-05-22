# Demo 12: Runtime Threat Detection with Falco

> **Duration:** ~10 minutes  
> **Section:** 5.3 — Runtime Monitoring and Incident Response  
> **Tools:** minikube, kubectl, Helm, Falco

---

## Learning Objectives

- Understand runtime security monitoring and why static scanning alone is not enough
- Install Falco in Kubernetes using Helm
- Trigger real security events (privilege escalation, file access, shell spawning)
- Read and interpret Falco alerts
- Understand how Falco integrates into an incident response workflow

---

## Prerequisites

- Helm installed: `helm version`

> **This demo requires a fresh minikube cluster.** Falco needs access to the Linux kernel to monitor syscalls. Modern Falco (0.37+) uses eBPF CO-RE which works on Linux kernel 5.8+ and Windows with WSL2 (kernel 5.15+).

### Platform Notes

| Platform | Status | Notes |
|----------|--------|-------|
| **Ubuntu 20.04+ / Linux** | Fully supported | Docker driver works |
| **Windows 11 + Docker Desktop** | Supported via WSL2 | WSL2 kernel 5.15+ |
| **Windows 10 + older WSL** | May not work | Use `--driver=virtualbox` if available |
| **macOS** | Not supported via Docker driver | eBPF probe loads but nested containerization prevents syscall capture; use Linux VM or cloud lab |

---

## Setup

### Start a dedicated minikube cluster for Falco

```bash
# Use a named profile to keep this isolated
minikube start \
  --profile=falco-demo \
  --cpus=2 \
  --memory=4096 \
  --driver=docker

kubectl config use-context falco-demo

mkdir -p ~/devsecops-demo/demo-12 && cd ~/devsecops-demo/demo-12
```

---

## Part 1: Install Falco

### Step 1 — Add the Falco Helm repository

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

### Step 2 — Install Falco with modern eBPF driver

```bash
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=modern_ebpf \
  --set tty=true

echo "Waiting for Falco to initialize (may take 2-3 minutes)..."
kubectl wait --for=condition=Ready pods --all -n falco --timeout=180s
echo "Falco is running."
```

### Step 3 — Verify Falco is monitoring

```bash
kubectl get pods -n falco
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=20
```

**Expected output:**
```
Falco version: 0.37.x
...
Starting health webserver with threadiness 4, listening on 0.0.0.0:8765
Enabled event sources: syscall
Opening 'syscall' source with modern BPF probe.
Loading rules file ...
```

---

## Part 2: Deploy a Test Workload

### Step 4 — Deploy a test pod

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

---

## Part 3: Trigger Security Events

Now we'll perform actions that are normal in a development environment but are red flags in production.

### Step 5 — Trigger: Shell spawned inside a container

```bash
echo "=== Triggering: Shell spawned in container ==="
kubectl exec -n falco-test test-workload -- bash -c "echo 'Attacker was here'"
```

**Falco rule triggered:** `Terminal shell in container`

```bash
# View Falco alert
sleep 3
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=20 | grep -A3 "shell\|Terminal" | head -20
```

**Expected Falco output:**
```
{"output":"10:05:00.123456789: Notice A shell was spawned in a container with an attached terminal
  (user=root user_loginuid=-1 k8s.ns=falco-test k8s.pod=test-workload container=test-workload
   shell=bash parent=runc cmdline=bash -c echo 'Attacker was here' ...)",
 "priority":"Notice",
 "rule":"Terminal shell in container",
 "source":"syscall",
 "tags":["container","mitre_execution","shell"]}
```

### Step 6 — Trigger: Write to sensitive directory

```bash
echo "=== Triggering: Write to /etc inside container ==="
kubectl exec -n falco-test test-workload -- bash -c "
  # Become root temporarily for this test (requires running as root for demo)
  id; ls /etc/passwd
" 2>/dev/null || true
```

### Step 7 — Trigger: Reading sensitive files

```bash
echo "=== Triggering: Read /etc/shadow equivalent ==="
kubectl exec -n falco-test test-workload -- bash -c "cat /etc/passwd"
```

**Falco rule triggered:** `Read sensitive file untrusted`

```bash
sleep 3
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=30 | grep -i "sensitive\|passwd\|shadow" | head -10
```

### Step 8 — Trigger: Network connection from container

```bash
echo "=== Triggering: Unexpected outbound connection ==="
kubectl exec -n falco-test test-workload -- bash -c "
  apt-get install -y curl -qq 2>/dev/null
  curl -s --max-time 3 http://example.com > /dev/null
" 2>/dev/null || true

sleep 3
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=30 | grep -i "network\|connect\|outbound" | head -10
```

---

## Part 4: View and Interpret Alerts

### Step 9 — View all recent Falco alerts with formatted output

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=100 | \
  grep '"rule"' | \
  python3 -c "
import sys, json

for line in sys.stdin:
    line = line.strip()
    try:
        entry = json.loads(line)
        priority = entry.get('priority', 'Unknown')
        rule = entry.get('rule', 'Unknown')
        source = entry.get('source', '')
        output = entry.get('output', '')
        
        # Extract the timestamp and key info
        timestamp = output.split(':')[0] if output else 'N/A'
        
        # Color coding by priority
        prefix = {'Emergency': '🚨', 'Critical': '🔴', 'Error': '🟠',
                  'Warning': '🟡', 'Notice': '🔵', 'Informational': '⚪'}.get(priority, '▪')
        
        print(f'{prefix} [{priority}] {rule}')
    except:
        pass
" 2>/dev/null | sort | uniq -c | sort -rn | head -20
```

**Expected output:**
```
      3 🔵 [Notice] Terminal shell in container
      2 🟡 [Warning] Read sensitive file untrusted
      1 🔵 [Notice] Contact K8S API Server From Container
```

### Step 10 — View a complete alert entry

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=100 | \
  grep '"Terminal shell"' | head -1 | python3 -m json.tool 2>/dev/null || \
  kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=50 | grep "shell" | head -3
```

---

## Part 5: Falco Rules (How Detection Works)

### Step 11 — View the built-in rules

```bash
kubectl exec -n falco -l app.kubernetes.io/name=falco -- \
  cat /etc/falco/falco_rules.yaml 2>/dev/null | head -60 || \
  echo "Rules are compiled into the Falco binary in this version."

# View the rules that are currently loaded
kubectl exec -n falco -l app.kubernetes.io/name=falco -- \
  falco --list 2>/dev/null | head -30 || true
```

### Step 12 — Understand a custom rule

```bash
cat > custom-rule-example.yaml << 'EOF'
# Custom Falco rule example — DO NOT apply, for reference only

# Macro: define reusable conditions
- macro: is_production_namespace
  condition: k8s.ns.name in (production, prod)

# List: define a set of allowed images
- list: approved_base_images
  items:
    - "nginx:1.25-alpine"
    - "python:3.11-alpine"

# Rule: detect unapproved images in production
- rule: Unapproved image in production
  desc: A container was started using an image not in the approved list
  condition: >
    container.started and
    is_production_namespace and
    not container.image.repository in (approved_base_images)
  output: >
    Unapproved image launched in production
    (image=%container.image.repository:%container.image.tag
     pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: WARNING
  tags: [container, supply-chain]
EOF

echo "Custom rule example saved."
echo "Deploy with: helm upgrade falco falcosecurity/falco --set-file customRules.custom-rules.yaml=custom-rule-example.yaml"
```

---

## Falco in Production: Alert Routing

```
Falco (runtime detection)
    │
    ▼
falcosidekick (alert router)
    ├──► Slack / Teams (immediate notification)
    ├──► PagerDuty / OpsGenie (on-call alerting)
    ├──► Elasticsearch / Splunk (SIEM integration)
    └──► Webhook (custom incident response)
```

Install falcosidekick:
```bash
# Add to helm install:
--set falcosidekick.enabled=true \
--set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/..."
```

---

## Cleanup

```bash
rm -rf ~/devsecops-demo/demo-12
kubectl delete namespace falco-test
minikube stop --profile=falco-demo
minikube delete --profile=falco-demo
kubectl config use-context minikube
```

---

## Key Takeaways

1. **Static scanning finds vulnerabilities in images — Falco detects attacks at runtime.** Both are needed.
2. Falco uses **eBPF to monitor syscalls** — it sees everything a process does without modifying the application.
3. Common attack indicators: shell spawned in container, writes to `/etc`, reading `/proc/*/environ`, outbound connections.
4. **Default rules catch most MITRE ATT&CK techniques** for containers — you get substantial coverage out of the box.
5. Route Falco alerts to your **SIEM or on-call system** via falcosidekick — raw logs on the node are not actionable at scale.
6. Falco complements (but does not replace) **NetworkPolicy, PSA, and RBAC** — it is your detection layer after prevention fails.
