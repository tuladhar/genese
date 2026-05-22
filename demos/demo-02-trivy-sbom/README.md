# Demo 02: Trivy Vulnerability Scanning & SBOM

> **Duration:** ~8 minutes  
> **Section:** 2.4 — Container Vulnerability Scanning  
> **Tools:** Trivy

---

## Learning Objectives

- Scan a container image for CVEs using Trivy
- Compare a vulnerable image against a minimal hardened image
- Generate a Software Bill of Materials (SBOM) with Trivy
- Use Trivy as a CI/CD security gate with exit codes

---

## Prerequisites

- Trivy installed (see [00-prerequisites.md](./00-prerequisites.md))
- Internet access (to pull images and download vulnerability databases)

---

## Setup

```bash
mkdir -p ~/devsecops-demo/demo-02 && cd ~/devsecops-demo/demo-02
```

---

## Part 1: Image Vulnerability Scanning

### Step 1 — Scan a known-vulnerable image

We use `python:3.8` — an older image with a significant number of CVEs. This is representative of what many teams still run in production.

```bash
trivy image python:3.8 --severity HIGH,CRITICAL
```

This will download Trivy's vulnerability database on first run (allow 30–60 seconds). Subsequent scans are fast.

**Sample output (truncated — exact numbers vary by Trivy DB version):**
```
python:3.8 (debian 12.x)
=========================
Total: 1600+ (HIGH: 1400+, CRITICAL: 180+)

┌─────────────────────────────┬────────────────┬──────────┬──────────────────────────────┬──────────────────────────────┐
│           Library           │ Vulnerability  │ Severity │      Installed Version       │        Fixed Version         │
├─────────────────────────────┼────────────────┼──────────┼──────────────────────────────┼──────────────────────────────┤
│ imagemagick                 │ CVE-2025-xxxxx │ CRITICAL │ 8:6.9.x                      │ 8:6.9.x+later               │
│ openssl                     │ CVE-2024-xxxxx │ HIGH     │ 3.x.x                        │ 3.x.x+later                  │
│ ...                         │ ...            │ ...      │ ...                          │ ...                          │
└─────────────────────────────┴────────────────┴──────────┴──────────────────────────────┴──────────────────────────────┘
```

### Step 2 — Scan a minimal image for comparison

```bash
trivy image python:3.11-alpine --severity HIGH,CRITICAL
```

**Expected output:**
```
python:3.11-alpine (alpine 3.x.x)
===================================
Total: 0 (HIGH: 0, CRITICAL: 0)   ← OS packages: clean
...
Total: 3 (HIGH: 3, CRITICAL: 0)   ← Python library packages: minimal
```

**Dramatically fewer CVEs** than the Debian-based image. Alpine's minimal OS and active patch cadence keep the attack surface small. This is why choosing the right base image matters.

### Step 3 — Save the scan as JSON (useful for CI/CD)

```bash
trivy image --format json --output python-3.8-scan.json python:3.8
echo "Scan saved to python-3.8-scan.json"
wc -l python-3.8-scan.json
```

Trivy's JSON output integrates into CI pipelines to fail builds that exceed a vulnerability threshold.

### Step 4 — Scan and fail on CRITICAL CVEs (CI/CD gate)

```bash
trivy image --exit-code 1 --severity CRITICAL python:3.8
echo "Exit code: $?"
```

**Expected output:**
```
Exit code: 1
```

With `--exit-code 1`, Trivy exits non-zero when matching vulnerabilities are found. This is the security gate in CI/CD.

---

## Part 2: SBOM Generation with Trivy

An **SBOM (Software Bill of Materials)** is an inventory of every package and library in your software. It lets you answer "am I affected by this new CVE?" without re-scanning every image — just re-scan the SBOM.

Trivy generates SBOMs natively — no additional tools needed.

### Step 5 — Generate an SBOM in SPDX format

```bash
trivy image --format spdx-json --output python-3.8-sbom.spdx.json python:3.8
echo "SBOM saved."
wc -l python-3.8-sbom.spdx.json
```

**Expected output:**
```
SBOM saved.
12847 python-3.8-sbom.spdx.json
```

### Step 6 — Inspect the SBOM

```bash
python3 -c "
import json
with open('python-3.8-sbom.spdx.json') as f:
    data = json.load(f)
packages = data.get('packages', [])
print(f'Total packages in SBOM: {len(packages)}')
print()
print('Sample packages:')
for p in packages[:5]:
    print(f\"  - {p.get('name', 'unknown')} {p.get('versionInfo', '')}\")
"
```

### Step 7 — Generate an SBOM in CycloneDX format

```bash
trivy image --format cyclonedx --output python-3.8-sbom.cdx.json python:3.8
echo "CycloneDX SBOM saved."
```

CycloneDX is the other widely-used SBOM standard. Both SPDX and CycloneDX are accepted by most compliance frameworks.

### Step 8 — Scan an existing SBOM for vulnerabilities

Once you have an SBOM, you can re-scan it without pulling the original image. This is useful when a new CVE is disclosed after the build:

```bash
trivy sbom python-3.8-sbom.spdx.json --severity HIGH,CRITICAL
```

**Expected output:**
```
python-3.8-sbom.spdx.json
==========================
Total: XX (HIGH: XX, CRITICAL: XX)
```

---

## Part 3: Scan Beyond Images

Trivy scans more than just container images.

### Step 9 — Scan a Dockerfile for misconfigurations

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

Trivy flags: `latest` tag, running as root, and hardcoded secret in ENV.

### Step 10 — Scan a Kubernetes manifest

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

**Expected output highlights:**
```
MEDIUM: Container 'app' of Pod 'insecure-pod' should set 'securityContext.allowPrivilegeEscalation' to false
HIGH:   Container 'app' of Pod 'insecure-pod' should not run as root
HIGH:   Container 'app' of Pod 'insecure-pod' should not be privileged
```

---

## Trivy Capabilities Summary

| What Trivy scans | Command |
|-----------------|---------|
| Container image CVEs | `trivy image <image>` |
| SBOM generation | `trivy image --format spdx-json` |
| Scan existing SBOM | `trivy sbom <file>` |
| Dockerfile misconfigs | `trivy config Dockerfile` |
| Kubernetes manifests | `trivy config pod.yaml` |
| Git repository secrets | `trivy repo .` |

---

## Cleanup

```bash
rm -rf ~/devsecops-demo/demo-02
```

---

## Key Takeaways

1. **Vulnerability scanning must be automated** — manual checks are never done consistently.
2. **Minimal images (Alpine-based)** have dramatically fewer CVEs than Debian/Ubuntu-based images.
3. **SBOM = inventory of your software** — generate it at build time, re-scan it whenever new CVEs are disclosed.
4. Use `--exit-code 1` in CI/CD to block deployments with critical vulnerabilities.
5. **Trivy scans everything**: images, SBOMs, Dockerfiles, Kubernetes manifests, Terraform, and git repos.
