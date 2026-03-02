---
name: trivy-sonarqube
description: Security scanning with Trivy and SonarQube — image, filesystem, IaC scanning, quality gates, CI integration
---

# Trivy & SonarQube — Skill Pack

Reference guide for integrating **vulnerability scanning and code quality gates** into every pipeline.

---

## 1. Trivy Overview

Trivy is a comprehensive scanner for:
- **Container images** — OS packages + language-specific dependencies
- **Filesystem** — `package-lock.json`, `pom.xml`, Go modules, etc.
- **IaC** — Kubernetes manifests, Helm charts, Terraform, Dockerfiles
- **SBOM** — Generate and scan Software Bill of Materials

---

## 2. Trivy — Container Image Scanning

```bash
# Scan an image — fail on CRITICAL and HIGH
trivy image --severity CRITICAL,HIGH --exit-code 1 \
  ghcr.io/org/my-app:abc123

# Scan with SBOM output
trivy image --format spdx-json --output sbom.json \
  ghcr.io/org/my-app:abc123

# Ignore unfixed vulnerabilities
trivy image --ignore-unfixed --severity CRITICAL \
  ghcr.io/org/my-app:abc123
```

### GitHub Actions Integration
```yaml
- name: Trivy image scan
  uses: aquasecurity/trivy-action@0.28.0
  with:
    image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
    format: table
    severity: CRITICAL,HIGH
    exit-code: 1
    # Optional — upload to GitHub Security tab
    # format: sarif
    # output: trivy-results.sarif

# - name: Upload scan results
#   uses: github/codeql-action/upload-sarif@v3
#   with:
#     sarif_file: trivy-results.sarif
```

---

## 3. Trivy — Filesystem Scanning (Dependencies)

```bash
# Scan project dependencies
trivy fs --severity CRITICAL,HIGH --exit-code 1 .

# Scan specific lockfile
trivy fs --severity CRITICAL,HIGH package-lock.json
trivy fs --severity CRITICAL,HIGH pom.xml
```

### GitHub Actions Integration
```yaml
- name: Trivy dependency scan
  uses: aquasecurity/trivy-action@0.28.0
  with:
    scan-type: fs
    scan-ref: .
    severity: CRITICAL,HIGH
    exit-code: 1
```

---

## 4. Trivy — IaC / Config Scanning

```bash
# Scan Kubernetes manifests
trivy config ./k8s/

# Scan Helm charts (rendered templates)
helm template my-app ./helm/my-app | trivy config --stdin

# Scan Dockerfiles
trivy config --file-patterns "Dockerfile" .

# Scan Terraform
trivy config ./terraform/
```

### GitHub Actions Integration
```yaml
- name: Trivy IaC scan
  uses: aquasecurity/trivy-action@0.28.0
  with:
    scan-type: config
    scan-ref: .
    severity: CRITICAL,HIGH,MEDIUM
    exit-code: 1
```

---

## 5. Trivy Configuration File

```yaml
# .trivy.yaml — project-level configuration
severity:
  - CRITICAL
  - HIGH

exit-code: 1

# Ignore specific CVEs with justification
ignore:
  - id: CVE-2023-XXXXX
    reason: "False positive — not reachable in our code path"
    expires: "2025-06-01"

# Skip specific directories
skip-dirs:
  - node_modules
  - .git
  - vendor

# Skip specific files
skip-files:
  - "**/*_test.go"
```

---

## 6. SonarQube Overview

SonarQube provides:
- **Static code analysis** — bugs, code smells, vulnerabilities
- **Security hotspots** — areas requiring manual security review
- **Code coverage tracking** — integrated with JaCoCo / Istanbul / lcov
- **Quality Gates** — pass/fail thresholds that block merges

---

## 7. SonarQube Quality Gate Configuration

### Recommended Quality Gate Conditions

| Metric | Operator | Value |
|--------|----------|-------|
| New Bugs | > | 0 |
| New Vulnerabilities | > | 0 |
| New Security Hotspots Reviewed | < | 100% |
| New Code Coverage | < | 80% |
| New Duplicated Lines (%) | > | 3% |
| Reliability Rating on New Code | worse than | A |
| Security Rating on New Code | worse than | A |

---

## 8. SonarQube — Node.js Integration

### sonar-project.properties
```properties
sonar.projectKey=org_my-node-app
sonar.projectName=My Node App
sonar.sources=src
sonar.tests=test
sonar.test.inclusions=**/*.test.ts,**/*.spec.ts
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.coverage.exclusions=**/*.test.ts,**/*.spec.ts,**/index.ts
sonar.eslint.reportPaths=eslint-report.json
```

### GitHub Actions
```yaml
- name: Run tests with coverage
  run: npm test -- --coverage

- name: SonarQube Scan
  uses: SonarSource/sonarqube-scan-action@v3
  env:
    SONAR_HOST_URL: ${{ secrets.SONAR_URL }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

- name: SonarQube Quality Gate
  uses: SonarSource/sonarqube-quality-gate-action@v1
  timeout-minutes: 5
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

---

## 9. SonarQube — Spring Boot Integration

### pom.xml Properties
```xml
<properties>
  <sonar.projectKey>org_my-spring-app</sonar.projectKey>
  <sonar.projectName>My Spring App</sonar.projectName>
  <sonar.sources>src/main/java</sonar.sources>
  <sonar.tests>src/test/java</sonar.tests>
  <sonar.java.coveragePlugin>jacoco</sonar.java.coveragePlugin>
  <sonar.coverage.jacoco.xmlReportPaths>
    target/site/jacoco/jacoco.xml
  </sonar.coverage.jacoco.xmlReportPaths>
  <sonar.coverage.exclusions>
    **/config/**,**/dto/**,**/entity/**,**/*Application.java
  </sonar.coverage.exclusions>
</properties>
```

### GitHub Actions
```yaml
- name: Build & test with JaCoCo
  run: ./mvnw verify -B

- name: SonarQube analysis
  run: >
    ./mvnw sonar:sonar
    -Dsonar.host.url=${{ secrets.SONAR_URL }}
    -Dsonar.token=${{ secrets.SONAR_TOKEN }}
```

---

## 10. Combined Trivy + SonarQube CI Pipeline

```yaml
name: Security & Quality
on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  security-events: write

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4

      - name: Trivy — dependency scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: fs
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Trivy — IaC scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: config
          scan-ref: .
          severity: CRITICAL,HIGH
          exit-code: 1

  sonarqube:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4
        with:
          fetch-depth: 0

      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@v3
        env:
          SONAR_HOST_URL: ${{ secrets.SONAR_URL }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

      - name: Quality Gate
        uses: SonarSource/sonarqube-quality-gate-action@v1
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

---

## 11. Triage & Remediation Checklist

| Finding | Action |
|---------|--------|
| CRITICAL CVE in base image | Update base image immediately |
| HIGH CVE in dependency | Update dependency; if no fix, evaluate workaround |
| IaC misconfiguration | Fix in template; add to CI checks |
| SonarQube blocker/critical | Fix before merge; no exceptions |
| Security hotspot | Review manually; mark as safe or fix |
| Unfixable CVE | Add to `.trivyignore` with expiry + justification |
