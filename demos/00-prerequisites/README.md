# Prerequisites & Environment Setup

> **Complete this before the training session.**  
> Allow **15–20 minutes** for installation and verification.

---

## 1. Docker Desktop

### Windows
1. Download from: https://www.docker.com/products/docker-desktop/
2. Run the installer — enable WSL2 integration when prompted.
3. Launch Docker Desktop and wait for the engine to start (green icon in taskbar).

### Ubuntu
```bash
# Remove old versions
sudo apt-get remove docker docker-engine docker.io containerd runc

# Install
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker

# Add your user to the docker group (log out and back in after)
sudo usermod -aG docker $USER
```

**Verify:**
```bash
docker run hello-world
```

---

## 2. Minikube

### Windows (Git Bash or PowerShell as Administrator)
```bash
# Using winget
winget install Kubernetes.minikube

# OR download directly
curl -Lo minikube.exe https://storage.googleapis.com/minikube/releases/latest/minikube-windows-amd64.exe
# Move minikube.exe to a directory in your PATH (e.g., C:\tools\)
```

### Ubuntu
```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/
```

**Verify:**
```bash
minikube version
```

---

## 3. kubectl

### Windows
```bash
winget install Kubernetes.kubectl
```

### Ubuntu
```bash
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

**Verify:**
```bash
kubectl version --client
```

---

## 4. Helm

### Windows
```bash
winget install Helm.Helm
```

### Ubuntu
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Verify:**
```bash
helm version --short
```

---

## 5. Trivy

### Windows (Git Bash)
```bash
# Using Scoop
scoop install trivy

# OR download binary from GitHub releases
# https://github.com/aquasecurity/trivy/releases/latest
# Extract trivy.exe and place in PATH
```

### Ubuntu
```bash
sudo apt-get install wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy
```

**Verify:**
```bash
trivy --version
```

---

## 6. Gitleaks

### Windows
```bash
# Using winget
winget install GitLeaks.GitLeaks

# OR Scoop
scoop install gitleaks
```

### Ubuntu
```bash
# Download latest binary
GITLEAKS_VERSION=$(curl -s "https://api.github.com/repos/gitleaks/gitleaks/releases/latest" | grep '"tag_name":' | sed 's/.*"v\([^"]*\)".*/\1/')
curl -sSfL "https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz" | tar -xz -C /tmp
sudo mv /tmp/gitleaks /usr/local/bin/
```

**Verify:**
```bash
gitleaks version
```

---

## 7. Full Environment Check

Run this script to verify everything is installed:

```bash
echo "=== DevSecOps Demo Environment Check ==="
echo ""

check() {
  if command -v $1 &>/dev/null; then
    echo "✓ $1: $($1 $2 2>&1 | head -1)"
  else
    echo "✗ $1: NOT FOUND"
  fi
}

check docker "--version"
check minikube "version"
check kubectl "version --client --short"
check helm "version --short"
check trivy "--version"
check gitleaks "version"

echo ""
echo "=== Docker Status ==="
docker info --format "Engine: {{.ServerVersion}}" 2>/dev/null || echo "✗ Docker daemon not running"
```

All tools should show a version number. Fix any that show `NOT FOUND` before the session.

---

## 8. Pre-Pull Docker Images

Run this to pre-download images used in the demos (saves time during the session):

```bash
docker pull python:3.8-slim
docker pull python:3.8-alpine
docker pull nginx:1.14.2
docker pull nginx:1.25-alpine
docker pull alpine:3.19
docker pull busybox:1.36
```

---

## 9. Start Your Default Minikube Cluster

Most demos use the default minikube cluster:

```bash
minikube start --cpus=2 --memory=4096 --driver=docker
```

> **Note:** The Network Policy demo (Demo 05) requires a separate cluster with Calico CNI.  
> The Falco demo (Demo 12) requires a separate cluster with specific settings.  
> Instructions for these are included in the respective demo files.

**Verify the cluster is running:**
```bash
kubectl get nodes
# Expected output:
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   Xm    v1.X.X
```
