# Demo 01: Docker Image Hardening

> **Duration:** ~10 minutes  
> **Section:** 2.2 — Docker Image Hardening  
> **Tools:** Docker

---

## Learning Objectives

- Identify security problems in a typical insecure Dockerfile
- Apply hardening techniques: minimal base image, non-root user, multi-stage build, read-only filesystem
- Measure the reduction in attack surface (image size + vulnerability count)

---

## Prerequisites

- Docker Desktop running
- Terminal open in the demo directory

---

## Setup

```bash
mkdir -p ~/devsecops-demo/demo-01 && cd ~/devsecops-demo/demo-01
```

---

## Part 1: The Insecure Image

### Step 1 — Write the insecure Dockerfile

This is a typical "it works on my machine" Dockerfile — the kind found in many real codebases.

```bash
cat > Dockerfile.insecure << 'EOF'
# BAD: Full OS base image — hundreds of unnecessary packages
FROM python:3.8

# BAD: No working directory, files land in /
COPY app.py .

# BAD: Installs curl and vim "just in case" — more attack surface
RUN apt-get update && apt-get install -y curl vim

# BAD: Runs as root
# (no USER directive means UID 0)

EXPOSE 8080
CMD ["python", "app.py"]
EOF
```

### Step 2 — Create the sample application

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

### Step 3 — Build the insecure image

```bash
docker build -f Dockerfile.insecure -t demo-app:insecure .
```

### Step 4 — Inspect what we have

```bash
# Check image size
docker images demo-app:insecure

# Verify the container runs as root
docker run --rm demo-app:insecure whoami
```

**Expected output:**
```
REPOSITORY    TAG        IMAGE ID       CREATED         SIZE
demo-app      insecure   abc123def456   5 seconds ago   921MB

root
```

The image is nearly **1 GB** and runs as root. A compromised process has full container privileges.

### Step 5 — Scan it

```bash
trivy image --severity HIGH,CRITICAL demo-app:insecure 2>/dev/null | tail -20
```

You will see dozens of HIGH and CRITICAL CVEs from the bloated base image.

---

## Part 2: The Hardened Image

### Step 6 — Write the hardened Dockerfile

```bash
cat > Dockerfile.hardened << 'EOF'
# Stage 1: Builder — install dependencies in a separate stage
FROM python:3.8-alpine AS builder

WORKDIR /app

# Copy only what's needed
COPY app.py .

# If there were dependencies: pip install --no-cache-dir -r requirements.txt

# Stage 2: Runtime — minimal image, no build tools
FROM python:3.8-alpine

# Set a non-root user and group
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only the application from the builder stage
COPY --from=builder /app/app.py .

# Set correct ownership
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

EXPOSE 8080

# Use exec form (not shell form) to handle signals correctly
CMD ["python", "app.py"]
EOF
```

### Step 7 — Build the hardened image

```bash
docker build -f Dockerfile.hardened -t demo-app:hardened .
```

### Step 8 — Compare sizes

```bash
docker images demo-app
```

**Expected output:**
```
REPOSITORY   TAG        IMAGE ID       CREATED          SIZE
demo-app     hardened   def456abc789   10 seconds ago   49.2MB
demo-app     insecure   abc123def456   2 minutes ago    921MB
```

The hardened image is **~95% smaller**.

### Step 9 — Verify non-root execution

```bash
docker run --rm demo-app:hardened whoami
```

**Expected output:**
```
appuser
```

### Step 10 — Run with additional runtime hardening flags

```bash
docker run -d \
  --name demo-hardened \
  --read-only \
  --cap-drop=ALL \
  --security-opt no-new-privileges:true \
  --tmpfs /tmp \
  -p 8888:8080 \
  demo-app:hardened

# Verify it's running
curl http://localhost:8888
```

**Expected output:**
```
Hello from DevSecOps Demo!
```

### Step 11 — Verify read-only filesystem enforcement

```bash
docker exec demo-hardened sh -c "echo test > /app/test.txt" 2>&1
```

**Expected output:**
```
sh: can't create /app/test.txt: Read-only file system
```

An attacker cannot write files to the container filesystem.

### Step 12 — Scan the hardened image

```bash
trivy image --severity HIGH,CRITICAL demo-app:hardened 2>/dev/null | tail -10
```

The CVE count is dramatically lower due to the minimal Alpine base.

---

## Hardening Techniques Summary

| Technique | Insecure | Hardened |
|-----------|----------|----------|
| Base image | `python:3.8` (Debian, ~921 MB) | `python:3.8-alpine` (~49 MB) |
| User | root (UID 0) | `appuser` (non-root) |
| Build tools in runtime | Yes (curl, vim) | No (multi-stage build) |
| Filesystem | Read-write | `--read-only` |
| Linux capabilities | All inherited | `--cap-drop=ALL` |
| Privilege escalation | Allowed | `--security-opt no-new-privileges` |

---

## Cleanup

```bash
docker stop demo-hardened
docker rm demo-hardened
docker rmi demo-app:insecure demo-app:hardened
rm -rf ~/devsecops-demo/demo-01
```

---

## Key Takeaways

1. **Minimal base images** reduce attack surface by eliminating hundreds of unused packages.
2. **Non-root users** prevent privilege escalation if a process is compromised.
3. **Multi-stage builds** ensure build tools never end up in production images.
4. **Read-only filesystems** and **dropped capabilities** provide runtime defense-in-depth.
5. Scan your images — an insecure base image from a public registry carries hundreds of CVEs.
