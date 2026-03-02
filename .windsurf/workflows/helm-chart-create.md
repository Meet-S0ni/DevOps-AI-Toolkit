---
description: Scaffold and publish a production-grade Helm chart from scratch
---

# Helm Chart Creation Workflow

Step-by-step workflow to create, validate, and publish a production-grade Helm chart.

---

## Prerequisites
- Helm 3 installed (`brew install helm` / `choco install kubernetes-helm`)
- `kubectl` configured for a test cluster
- OCI registry for chart publishing (GHCR, ECR, or ChartMuseum)

---

## Step 1 — Scaffold the chart

```bash
# Create chart skeleton
helm create helm/my-app

# Clean up defaults — remove unnecessary files
rm -rf helm/my-app/templates/tests
rm helm/my-app/templates/hpa.yaml            # we'll create our own
rm helm/my-app/templates/NOTES.txt

# Verify structure
tree helm/my-app/
```

Expected structure:
```
helm/my-app/
├── Chart.yaml
├── values.yaml
├── .helmignore
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    └── ingress.yaml
```

---

## Step 2 — Configure Chart.yaml

```yaml
# helm/my-app/Chart.yaml
apiVersion: v2
name: my-app
description: Production-grade chart for my-app
type: application
version: 1.0.0
appVersion: "1.0.0"
maintainers:
  - name: DevOps Team
    email: devops@example.com
keywords:
  - api
  - microservice
home: https://github.com/org/my-app
sources:
  - https://github.com/org/my-app
```

---

## Step 3 — Define values.yaml with security defaults

```yaml
# helm/my-app/values.yaml
# See skills/kubernetes-helm/SKILL.md for the full production values template
# Key sections to include:

replicaCount: 2

image:
  repository: ghcr.io/org/my-app
  tag: ""
  pullPolicy: IfNotPresent

serviceAccount:
  create: true
  annotations: {}

podSecurityContext:
  runAsNonRoot: true
  fsGroup: 1000

securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: "50%"

networkPolicy:
  enabled: true
```

---

## Step 4 — Add missing templates

### HPA
```bash
cat > helm/my-app/templates/hpa.yaml << 'EOF'
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
EOF
```

### PDB
```bash
cat > helm/my-app/templates/pdb.yaml << 'EOF'
{{- if .Values.pdb.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  minAvailable: {{ .Values.pdb.minAvailable | quote }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
{{- end }}
EOF
```

### Network Policy
```bash
cat > helm/my-app/templates/networkpolicy.yaml << 'EOF'
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "my-app.fullname" . }}
spec:
  podSelector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}
      ports:
        - port: {{ .Values.service.port }}
  egress:
    - to: []
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
{{- end }}
EOF
```

---

## Step 5 — Add values schema validation

```bash
cat > helm/my-app/values.schema.json << 'EOF'
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["image", "replicaCount"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": { "type": "string" },
        "tag": { "type": "string" },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"]
        }
      }
    },
    "resources": {
      "type": "object",
      "properties": {
        "requests": { "type": "object" },
        "limits": { "type": "object" }
      }
    }
  }
}
EOF
```

---

## Step 6 — Create environment-specific overrides

```bash
# Staging overrides
cat > helm/my-app/values-staging.yaml << 'EOF'
replicaCount: 1

autoscaling:
  enabled: false

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi
EOF

# Production overrides
cat > helm/my-app/values-prod.yaml << 'EOF'
replicaCount: 3

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 15

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi
EOF
```

---

## Step 7 — Validate the chart

```bash
# Lint with strict mode
helm lint helm/my-app --strict

# Render templates locally
helm template my-app helm/my-app

# Render with prod values
helm template my-app helm/my-app -f helm/my-app/values-prod.yaml

# Dry-run against cluster
helm upgrade --install my-app helm/my-app \
  --namespace test --dry-run

# Validate rendered manifests with kubeconform
helm template my-app helm/my-app | kubeconform -strict

# Score manifests with kube-score
helm template my-app helm/my-app | kube-score score -
```

---

## Step 8 — Package and publish

### Option A — OCI registry (GHCR)
```bash
# Login to registry
echo $GITHUB_TOKEN | helm registry login ghcr.io -u $GITHUB_ACTOR --password-stdin

# Package
helm package helm/my-app

# Push
helm push my-app-1.0.0.tgz oci://ghcr.io/org/charts

# Install from OCI
helm install my-app oci://ghcr.io/org/charts/my-app --version 1.0.0
```

### Option B — ChartMuseum
```bash
# Add repo
helm repo add my-charts https://charts.example.com

# Package and push
helm package helm/my-app
curl --data-binary "@my-app-1.0.0.tgz" https://charts.example.com/api/charts

# Install
helm repo update
helm install my-app my-charts/my-app --version 1.0.0
```

---

## Checklist

- [ ] Chart scaffolded with `helm create`
- [ ] `Chart.yaml` has version, appVersion, maintainers
- [ ] `values.yaml` has security defaults (non-root, read-only, drop ALL)
- [ ] All templates include: deployment, service, SA, HPA, PDB, network policy
- [ ] `values.schema.json` validates required fields
- [ ] Environment-specific values files created
- [ ] `helm lint --strict` passes
- [ ] `helm template` renders without errors
- [ ] `kubeconform` validates manifests
- [ ] Chart packaged and pushed to OCI registry
