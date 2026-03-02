---
description: CD deployment to Kubernetes via Helm — canary, rollback, namespace strategy
---

# CD Deployment Workflow

Step-by-step workflow to deploy applications to Kubernetes using Helm with progressive delivery.

---

## Prerequisites
- Kubernetes cluster with `kubectl` access
- Helm 3 installed
- CI pipeline publishing images (see `/ci-pipeline`)
- OIDC configured for GitHub → Cloud provider auth
- Namespaces pre-created with quotas and network policies

---

## Steps

### 1. Prepare environment-specific values

```bash
# Create per-environment value overrides
helm/my-app/
├── values.yaml          # defaults
├── values-dev.yaml      # dev overrides
├── values-staging.yaml  # staging overrides
└── values-prod.yaml     # prod overrides
```

```yaml
# values-prod.yaml
replicaCount: 3

image:
  repository: ghcr.io/org/my-app
  tag: ""   # set by CI

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 15

ingress:
  enabled: true
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts: [api.example.com]
```

### 2. Create the CD workflow

```yaml
# .github/workflows/cd.yml
name: CD
on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

concurrency:
  group: deploy-${{ github.ref_name }}
  cancel-in-progress: false   # NEVER cancel in-flight deploys

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4

      - name: OIDC auth
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Configure kubectl
        run: aws eks update-kubeconfig --name staging-cluster

      - name: Helm deploy to staging
        run: |
          helm upgrade --install my-app ./helm/my-app \
            --namespace my-app-staging \
            --atomic --timeout 5m \
            --set image.tag=${{ github.sha }} \
            -f helm/my-app/values-staging.yaml

      - name: Smoke test
        run: |
          kubectl wait --for=condition=available deployment/my-app \
            --namespace my-app-staging --timeout=120s
          # Hit health endpoint
          STAGING_URL=$(kubectl get svc my-app -n my-app-staging -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          curl -sf "http://${STAGING_URL}/healthz" || exit 1

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production   # requires manual approval
    steps:
      - uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4

      - name: OIDC auth
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_PROD }}
          aws-region: us-east-1

      - name: Configure kubectl
        run: aws eks update-kubeconfig --name prod-cluster

      - name: Helm deploy to production
        run: |
          helm upgrade --install my-app ./helm/my-app \
            --namespace my-app-prod \
            --atomic --timeout 5m \
            --history-max 5 \
            --set image.tag=${{ github.sha }} \
            -f helm/my-app/values-prod.yaml

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/my-app \
            --namespace my-app-prod --timeout=180s
```

### 3. Set up GitHub Environments

1. **Settings → Environments → New environment**
2. Create `staging` (no approval required)
3. Create `production`:
   - ✅ Required reviewers (add team leads)
   - ✅ Wait timer (optional, e.g. 5 min)
   - Add secrets: `AWS_ROLE_ARN_PROD`

### 4. Namespace preparation (one-time)

```bash
# Create namespaces with quotas
kubectl create namespace my-app-staging
kubectl create namespace my-app-prod

# Apply resource quotas
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: my-app-prod
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "30"
EOF

# Apply default-deny network policy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: my-app-prod
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF
```

### 5. Manual rollback procedure

```bash
# View release history
helm history my-app --namespace my-app-prod

# Rollback to previous release
helm rollback my-app 0 --namespace my-app-prod

# Rollback to specific revision
helm rollback my-app 3 --namespace my-app-prod

# Verify rollback
kubectl rollout status deployment/my-app --namespace my-app-prod
```

### 6. Canary deployment (advanced)

```yaml
# Using Argo Rollouts for canary
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }     # observe 10% traffic
        - setWeight: 30
        - pause: { duration: 5m }
        - setWeight: 60
        - pause: { duration: 5m }
        - setWeight: 100
      canaryService: my-app-canary
      stableService: my-app-stable
      trafficRouting:
        nginx:
          stableIngress: my-app-ingress
```
