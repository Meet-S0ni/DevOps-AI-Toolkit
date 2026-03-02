---
description: End-to-end CI pipeline — build, test, lint, scan, publish image
---

# CI Pipeline Setup

Step-by-step workflow to set up a complete CI pipeline with GitHub Actions.

---

## Prerequisites
- GitHub repository with branch protection on `main`
- Container registry (GHCR recommended)
- Trivy and SonarQube configured (see `/security-scan` workflow)

---

## Steps

### 1. Create the CI workflow file

```bash
mkdir -p .github/workflows
touch .github/workflows/ci.yml
```

### 2. Define triggers and permissions

```yaml
name: CI
on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

permissions:
  contents: read
  pull-requests: write
  packages: write
  security-events: write

concurrency:
  group: ci-${{ github.head_ref || github.ref_name }}
  cancel-in-progress: true
```

### 3. Add lint & test job

**Node.js:**
```yaml
jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/
```

**Spring Boot:**
```yaml
jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
          cache: maven
      - run: ./mvnw verify -B
      - uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: target/surefire-reports/
```

### 4. Add security scanning job

```yaml
  security-scan:
    runs-on: ubuntu-latest
    needs: lint-test
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
          severity: CRITICAL,HIGH
          exit-code: 1
```

### 5. Add SonarQube analysis job

```yaml
  sonarqube:
    runs-on: ubuntu-latest
    needs: lint-test
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4
        with:
          fetch-depth: 0
      - uses: SonarSource/sonarqube-scan-action@v3
        env:
          SONAR_HOST_URL: ${{ secrets.SONAR_URL }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      - uses: SonarSource/sonarqube-quality-gate-action@v1
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### 6. Add Docker build & push job

```yaml
  build-image:
    runs-on: ubuntu-latest
    needs: [security-scan, sonarqube]
    if: github.event_name == 'push'   # only on main merge
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Trivy — image scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: 1
```

### 7. Configure branch protection

1. Go to **Settings → Branches → Add rule** for `main`
2. Enable:
   - ✅ Require pull request reviews (≥ 1 approval)
   - ✅ Require status checks: `lint-test`, `security-scan`, `sonarqube`
   - ✅ Require branches to be up to date
   - ✅ Require signed commits (optional but recommended)

### 8. Verify

```bash
# Create a test PR and confirm all jobs pass
git checkout -b test/ci-pipeline
echo "test" >> test-ci.txt
git add . && git commit -m "ci: test pipeline"
git push origin test/ci-pipeline
# Open PR in GitHub and verify all checks run
```
