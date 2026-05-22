# Demo 11: Secret Scanning with Gitleaks

> **Duration:** ~6 minutes  
> **Section:** 4.5 — Secret Scanning and Detection  
> **Tools:** Gitleaks, git

---

## Learning Objectives

- Detect hardcoded secrets in a git repository using Gitleaks
- Understand how secrets end up in code (even when "removed")
- Configure Gitleaks as a pre-commit hook to prevent secrets from being committed
- Create a baseline to suppress known false positives

---

## Prerequisites

- Gitleaks installed: `gitleaks version`
- Git installed: `git --version`

---

## Setup

```bash
mkdir -p ~/devsecops-demo/demo-11 && cd ~/devsecops-demo/demo-11
```

---

## Part 1: Detect Secrets in a Repository

### Step 1 — Create a sample repository with secrets

```bash
mkdir -p demo-repo && cd demo-repo
git init
git config user.email "demo@dangote.com"
git config user.name "Demo User"
```

### Step 2 — Create files that contain secrets

```bash
# Commit 1: "innocent" config file
cat > config.py << 'EOF'
# Application configuration
DATABASE_HOST = "postgres.internal.example.com"
DATABASE_PORT = 5432
DATABASE_NAME = "production_db"

# TODO: move these to environment variables
DATABASE_USER = "admin"
DATABASE_PASSWORD = "p@ssw0rd!SuperSecret2024"
EOF

git add config.py
git commit -m "Add database configuration"

# Commit 2: Add API keys
cat > .env << 'EOF'
# API credentials — hardcoded for "testing"
STRIPE_SECRET_KEY=sk_test_zQPyAkXOPdh2IEbhCDN0gZhMbJ83XYZABC12345
AWS_SECRET_ACCESS_KEY=P9Tq/ABcDefGHiJkL+MnOpQrStUvWxYzABcDeF12
GITHUB_TOKEN=ghp_ABcDefGHiJkLmNoPqRsTuVwXyZABcDeFgHiJ12
EOF

git add .env
git commit -m "Add API credentials config"

# Commit 3: "Remove" the secrets (but they're still in git history!)
cat > .env << 'EOF'
# Secrets moved to secrets manager
STRIPE_SECRET_KEY=
AWS_SECRET_ACCESS_KEY=
GITHUB_TOKEN=
EOF

git add api_client.py
git commit -m "Fix: move secrets to environment variables"
```

### Step 3 — Run Gitleaks to detect secrets

```bash
cd ~/devsecops-demo/demo-11/demo-repo

echo "=== Scanning entire git history ==="
gitleaks detect --source . --verbose 2>&1
```

**Expected output:**
```
    ○
    │╲
    │ ○
    ○ ░
    ░    gitleaks

Finding:     ...AKIAIOSFODNN7EXAMPLE...
Secret:      AKIAIOSFODNN7EXAMPLE
RuleID:      aws-access-token
Entropy:     3.58
File:        api_client.py
Line:        4
Commit:      abc123def
Author:      Demo User
Email:       demo@dangote.com
Date:        2025-01-01T10:00:00Z
Fingerprint: abc123...

...

4 secrets detected.
```

Gitleaks found **4 secrets across the git history** — including ones in commits that have been "cleaned up." Removing a secret from the latest commit does NOT remove it from git history.

### Step 4 — Scan only the current working tree (no git history)

```bash
gitleaks detect --source . --no-git --verbose 2>&1
```

This scans the current files without git history — useful for scanning individual files before committing.

---

## Part 2: Understand the Impact

### Step 5 — Show that "removed" secrets are still accessible

```bash
echo "=== The 'removed' credentials are still in git history ==="

# Find the commit where the credentials were added
git log --oneline

# Get the credentials from the commit that added secrets
COMMIT=$(git log --oneline | grep "API credentials" | awk '{print $1}')
git show $COMMIT:.env | grep -E "KEY|TOKEN|SECRET"
```

**Expected output:**
```
STRIPE_SECRET_KEY=sk_test_zQPyAkXOPdh2IEbhCDN0gZhMbJ83XYZABC12345
AWS_SECRET_ACCESS_KEY=P9Tq/ABcDefGHiJkL+MnOpQrStUvWxYzABcDeF12
GITHUB_TOKEN=ghp_ABcDefGHiJkLmNoPqRsTuVwXyZABcDeFgHiJ12
```

The credentials are permanently in git history. The correct response is to **revoke and rotate** the credentials, not just remove them from code.

---

## Part 3: Pre-commit Hook (Prevention)

### Step 6 — Install Gitleaks as a pre-commit hook

```bash
# Create the hooks directory
mkdir -p .git/hooks

# Create the pre-commit hook
cat > .git/hooks/pre-commit << 'HOOK'
#!/bin/sh
# Scan staged files for secrets before allowing the commit

echo "Running Gitleaks secret scan..."

gitleaks protect --staged --verbose

if [ $? -ne 0 ]; then
    echo ""
    echo "COMMIT BLOCKED: Gitleaks detected secrets in staged changes."
    echo "Remove the secrets, then commit again."
    echo "If this is a false positive, add it to .gitleaks.toml baseline."
    exit 1
fi

echo "No secrets detected. Commit allowed."
HOOK

chmod +x .git/hooks/pre-commit
echo "Pre-commit hook installed."
```

### Step 7 — Test the pre-commit hook

```bash
# Try to commit a file with a secret
cat > new-secret.py << 'EOF'
# Oops, forgot to use env var
STRIPE_SECRET_KEY = "sk_test_zQPyAkXOPdh2IEbhCDN0gZhMbJ83XYZABC12345"
EOF

git add new-secret.py
git commit -m "Add payment integration"
```

**Expected output:**
```
Running Gitleaks secret scan...

Finding:     ...STRIPE_SECRET_KEY = "sk_live_zQPy...
RuleID:      stripe-access-token
File:        new-secret.py
Line:        2

COMMIT BLOCKED: Gitleaks detected secrets in staged changes.
Remove the secrets, then commit again.
```

The commit is **blocked before the secret enters git history**.

---

## Part 4: Handle False Positives with a Baseline

### Step 8 — Create a baseline for known non-sensitive patterns

```bash
cat > .gitleaks.toml << 'EOF'
title = "Dangote DevSecOps Demo - Gitleaks Config"

[extend]
# Use the default ruleset
useDefault = true

[[rules]]
description = "Custom: Dangote internal API keys (not real secrets)"
id = "dangote-internal"
# Add any project-specific patterns here

[allowlist]
description = "Known false positives"
paths = [
  # Test files that intentionally contain example credentials
  "tests/fixtures/.*",
  "docs/examples/.*",
]
regexes = [
  # Example/placeholder credentials in documentation
  "EXAMPLE_KEY",
  "YOUR_API_KEY_HERE",
  "placeholder",
]
stopwords = [
  "example",
  "placeholder",
  "test",
  "fake",
  "dummy",
]
EOF

echo "Gitleaks config written to .gitleaks.toml"
```

---

## CI/CD Integration

In a CI/CD pipeline (GitHub Actions example):

```yaml
# .github/workflows/secret-scan.yml
name: Secret Scan
on: [push, pull_request]

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0    # Full history required
    - uses: gitleaks/gitleaks-action@v2
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Cleanup

```bash
cd ~/devsecops-demo/demo-11
rm -rf demo-repo
rm -rf ~/devsecops-demo/demo-11
```

---

## Key Takeaways

1. **Removing a secret from code does not remove it from git history** — always rotate credentials that were ever committed.
2. **Gitleaks scans the full git history** — secrets committed months ago are still detectable (and exploitable).
3. **Pre-commit hooks are the earliest prevention** — stop secrets before they ever enter the repository.
4. **CI/CD scanning is a safety net** — catches secrets that slip past local hooks (e.g., from other team members).
5. When a secret is found in git history: **1) Revoke it immediately, 2) Rotate it, 3) Then clean the git history** (with `git filter-repo`).
