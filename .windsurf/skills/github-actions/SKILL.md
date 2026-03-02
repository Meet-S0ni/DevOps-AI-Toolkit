---
name: github-actions
description: CI/CD pipeline patterns with GitHub Actions — build, test, scan, deploy, reusable workflows, OIDC
---

# GitHub Actions — Skill Pack

Reference guide for building **secure, maintainable CI/CD pipelines** with GitHub Actions.

---

## 1. Repository Structure

```
.github/
├── workflows/
│   ├── ci.yml              # PR checks: lint, test, scan
│   ├── cd.yml              # Deploy to K8s on main merge
│   └── reusable-build.yml  # Reusable build workflow
├── actions/
│   └── setup-tools/        # Composite action for tool setup
│       └── action.yml
└── CODEOWNERS
```

---

## 2. CI Pipeline Template (Node.js)

```yaml
name: CI
on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

concurrency:
  group: ci-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4
      - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f8  # v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage
      - uses: actions/upload-artifact@5d5d22a31266ced268874388b861e4b58bb5c2f3  # v4
        with:
          name: coverage
          path: coverage/

  security-scan:
    runs-on: ubuntu-latest
    needs: lint-test
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4
      - name: Trivy filesystem scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: fs
          severity: CRITICAL,HIGH
          exit-code: 1

  build-image:
    runs-on: ubuntu-latest
    needs: security-scan
    permissions:
      packages: write
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
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Trivy image scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: 1
```

---

## 3. CI Pipeline Template (Spring Boot)

```yaml
name: CI — Spring Boot
on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  build-test:
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

  sonarqube:
    runs-on: ubuntu-latest
    needs: build-test
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4
        with:
          fetch-depth: 0  # full history for blame
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
          cache: maven
      - run: >
          ./mvnw sonar:sonar
          -Dsonar.host.url=${{ secrets.SONAR_URL }}
          -Dsonar.token=${{ secrets.SONAR_TOKEN }}
          -Dsonar.projectKey=${{ github.repository_owner }}_${{ github.event.repository.name }}
```

---

## 4. CD Deploy Template (Helm to Kubernetes)

```yaml
name: CD
on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write   # OIDC

concurrency:
  group: deploy-prod
  cancel-in-progress: false   # never cancel in-flight deploys

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production    # requires manual approval
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4

      # OIDC auth to cloud provider
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy
          aws-region: us-east-1

      - name: Configure kubeconfig
        run: aws eks update-kubeconfig --name my-cluster --region us-east-1

      - name: Helm deploy
        run: |
          helm upgrade --install my-app ./helm/my-app \
            --namespace my-app-prod \
            --atomic \
            --timeout 5m \
            --set image.tag=${{ github.sha }} \
            --values helm/my-app/values-prod.yaml
```

---

## 5. Reusable Workflows

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Docker Build
on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
      dockerfile:
        required: false
        type: string
        default: Dockerfile
    secrets:
      registry-password:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.registry-password }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          file: ${{ inputs.dockerfile }}
          tags: ghcr.io/${{ github.repository_owner }}/${{ inputs.image-name }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

```yaml
# Calling the reusable workflow
jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      image-name: api
    secrets:
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

---

## 6. Security Best Practices Checklist

| Practice | How |
|----------|-----|
| Pin actions to SHA | `uses: actions/checkout@<sha>` |
| Minimal permissions | `permissions: contents: read` at top level |
| OIDC for cloud auth | `id-token: write` + provider trust policy |
| Concurrency control | `concurrency:` block to prevent race conditions |
| No secrets in logs | Never `echo ${{ secrets.* }}`; use masking |
| Env-based approvals | `environment: production` with required reviewers |
| Immutable tags | Tag with `${{ github.sha }}`; never `latest` |
| Cache effectively | `cache: npm` / `cache: maven` + Docker GHA cache |

---

## 7. Matrix Strategy Example

```yaml
jobs:
  test:
    strategy:
      matrix:
        node-version: [18, 20, 22]
        os: [ubuntu-latest]
      fail-fast: true
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4
      - uses: actions/setup-node@60edb5dd545a775178f52524783378180af0d1f8  # v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci && npm test
```

---

## 8. Composite Actions

```yaml
# .github/actions/setup-tools/action.yml
name: Setup DevOps Tools
description: Install Helm, kubectl, Trivy
runs:
  using: composite
  steps:
    - name: Install Helm
      shell: bash
      run: |
        curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    - name: Install kubectl
      shell: bash
      run: |
        curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
        chmod +x kubectl && sudo mv kubectl /usr/local/bin/
    - name: Install Trivy
      shell: bash
      run: |
        sudo apt-get install -y wget apt-transport-https gnupg lsb-release
        wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
        echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/trivy.list
        sudo apt-get update && sudo apt-get install -y trivy
```

---

## 9. AI-DLC Pipeline Patterns

Align CI/CD pipelines with the **three AI-DLC phases** for maximum safety and traceability.

### Phase-Aware Pipeline Architecture

```
  Inception Phase           Construction Phase          Operations Phase
  ─────────────            ──────────────────          ────────────────
  (No CI needed)           PR → CI Pipeline            Main → CD Pipeline
                                 │                           │
                           ┌─────┴──────┐             ┌─────┴──────┐
                           │ lint-test   │             │ deploy-stg │
                           │ scan       │             │ verify-stg │
                           │ sonarqube   │             │ gate-check │
                           │ build-image │             │ deploy-prod│
                           └────────────┘             │ verify-prod│
                                                       └────────────┘
```

### Context Propagation in Pipelines

Pass **AI-DLC context** (Unit, Bolt, phase) through CI/CD for traceability:

```yaml
env:
  AIDLC_UNIT: ${{ github.event.pull_request.title }}
  AIDLC_BOLT: ${{ github.run_number }}
  AIDLC_PHASE: construction
  AIDLC_SHA: ${{ github.sha }}

# Tag images with context
tags: |
  ghcr.io/${{ github.repository }}:${{ github.sha }}
  ghcr.io/${{ github.repository }}:bolt-${{ github.run_number }}
```

### Automated Gate Checks

```yaml
  gate-check:
    runs-on: ubuntu-latest
    needs: [build-image]
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4

      - name: Verify AI-DLC gate compliance
        run: |
          echo "=== Construction → Operations Gate ==="

          # 1. Trivy scan passed (checked in previous job)
          echo "✅ Trivy scan: passed"

          # 2. SonarQube quality gate passed
          echo "✅ SonarQube: passed"

          # 3. Helm lint
          helm lint ./helm/${{ github.event.repository.name }} --strict
          echo "✅ Helm lint: passed"

          # 4. Verify security context in Helm templates
          helm template test ./helm/${{ github.event.repository.name }} | \
            grep -q "runAsNonRoot: true" && echo "✅ Non-root: configured" || \
            (echo "❌ Missing runAsNonRoot" && exit 1)

          # 5. Verify PDB exists
          helm template test ./helm/${{ github.event.repository.name }} | \
            grep -q "PodDisruptionBudget" && echo "✅ PDB: configured" || \
            echo "⚠️ PDB not found (check if replicas > 1)"

          echo "=== Gate: PASSED ==="
```

### Post-Deploy Verification Job

```yaml
  verify-deployment:
    runs-on: ubuntu-latest
    needs: deploy
    steps:
      - name: Wait for rollout
        run: kubectl rollout status deployment/${{ env.APP }} -n ${{ env.NAMESPACE }} --timeout=180s

      - name: Health check
        run: |
          ENDPOINT=$(kubectl get svc ${{ env.APP }} -n ${{ env.NAMESPACE }} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          for i in {1..10}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" "http://${ENDPOINT}/healthz")
            if [ "$STATUS" = "200" ]; then
              echo "✅ Health check passed"
              exit 0
            fi
            sleep 10
          done
          echo "❌ Health check failed"
          exit 1

      - name: Log Bolt completion
        if: success()
        run: |
          echo "Bolt ${{ github.run_number }} deployed successfully" >> .ai-dlc/bolts/bolt-${{ github.run_number }}.md
```
