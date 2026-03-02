---
description: Run Trivy + SonarQube security scans locally or in CI and interpret results
---

# Security Scan Workflow

Step-by-step workflow to run comprehensive security scans using Trivy and SonarQube.

---

## Part A — Trivy (Local)

### 1. Install Trivy

```bash
# macOS
brew install trivy

# Linux (Debian/Ubuntu)
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy

# Windows (Scoop)
scoop install trivy
```

### 2. Scan project dependencies

```bash
# Scan the entire project filesystem
trivy fs --severity CRITICAL,HIGH .

# Scan specific lockfiles
trivy fs --severity CRITICAL,HIGH package-lock.json   # Node.js
trivy fs --severity CRITICAL,HIGH pom.xml             # Spring Boot
```

### 3. Scan container images

```bash
# Build then scan
docker build -t my-app:local .
trivy image --severity CRITICAL,HIGH my-app:local

# Scan a remote image
trivy image --severity CRITICAL,HIGH ghcr.io/org/my-app:abc123
```

### 4. Scan IaC (Kubernetes / Helm / Terraform)

```bash
# Scan K8s manifests
trivy config ./k8s/

# Scan Helm charts (render then scan)
helm template my-app ./helm/my-app | trivy config --stdin

# Scan Terraform
trivy config ./terraform/

# Scan Dockerfiles
trivy config .
```

### 5. Create ignore file for false positives

```bash
# .trivyignore
# CVE-2023-XXXXX — false positive, not reachable (expires 2025-06-01)
CVE-2023-XXXXX

# CVE-2024-YYYYY — accepted risk, no fix available
CVE-2024-YYYYY
```

### 6. Interpret Trivy results

| Severity | Action Required |
|----------|----------------|
| CRITICAL | **Fix immediately** — block deployment |
| HIGH     | **Fix before merge** — block PR |
| MEDIUM   | Fix within sprint — track in backlog |
| LOW      | Fix opportunistically |

---

## Part B — SonarQube (Local)

### 1. Start SonarQube locally

```bash
# Docker Compose
docker run -d --name sonarqube \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:community

# Wait for startup, then open http://localhost:9000
# Default credentials: admin / admin (change immediately)
```

### 2. Create project and generate token

1. Log in to SonarQube → **Projects → Create Project**
2. Set project key and name
3. Go to **My Account → Security → Generate Token**
4. Save the token securely

### 3. Run SonarQube analysis

**Node.js:**
```bash
# Install scanner
npm install -g sonarqube-scanner

# Create sonar-project.properties (see skills/trivy-sonarqube)

# Run scanner
npx sonarqube-scanner \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=YOUR_TOKEN
```

**Spring Boot:**
```bash
./mvnw verify sonar:sonar \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=YOUR_TOKEN
```

### 4. Review results

1. Open SonarQube dashboard → your project
2. Check: **Bugs**, **Vulnerabilities**, **Security Hotspots**, **Code Smells**
3. Review **Quality Gate** status — must be **Passed**
4. Address all **Security Hotspots** — review and mark as Safe or Fix

### 5. Configure Quality Gate

1. **Quality Gates → Create**
2. Add conditions:
   - New Bugs = 0
   - New Vulnerabilities = 0
   - New Coverage ≥ 80%
   - New Duplicated Lines ≤ 3%
   - Security Rating = A
3. Set as default gate

---

## Part C — CI Integration

### Combined GitHub Actions job

```yaml
# See workflows/ci-pipeline.md for full pipeline
# Quick reference:
security-scan:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Trivy FS
      uses: aquasecurity/trivy-action@0.28.0
      with:
        scan-type: fs
        severity: CRITICAL,HIGH
        exit-code: 1
    - name: Trivy Config
      uses: aquasecurity/trivy-action@0.28.0
      with:
        scan-type: config
        severity: CRITICAL,HIGH
        exit-code: 1
    - name: SonarQube
      uses: SonarSource/sonarqube-scan-action@v3
      env:
        SONAR_HOST_URL: ${{ secrets.SONAR_URL }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    - name: Quality Gate
      uses: SonarSource/sonarqube-quality-gate-action@v1
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

---

## Checklist

- [ ] Trivy installed locally
- [ ] `.trivyignore` created for known false positives
- [ ] SonarQube running (local or hosted)
- [ ] `sonar-project.properties` configured
- [ ] Quality Gate set with correct thresholds
- [ ] CI pipeline includes both Trivy and SonarQube jobs
- [ ] Branch protection requires security-scan job to pass
