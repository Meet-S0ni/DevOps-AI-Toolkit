---
name: kubernetes-helm
description: Kubernetes manifest & Helm chart authoring — security contexts, probes, HPA, network policies, Helm best practices
---

# Kubernetes & Helm — Skill Pack

Reference guide for writing **production-grade Kubernetes manifests and Helm charts**.

---

## 1. Helm Chart Structure

```
helm/my-app/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
├── values.schema.json        # JSON schema for values validation
├── templates/
│   ├── _helpers.tpl           # Template helpers
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── networkpolicy.yaml
│   ├── serviceaccount.yaml
│   ├── configmap.yaml
│   └── tests/
│       └── test-connection.yaml
└── .helmignore
```

---

## 2. Chart.yaml Template

```yaml
apiVersion: v2
name: my-app
description: A production-grade Helm chart for my-app
type: application
version: 1.0.0        # chart version — bump on chart changes
appVersion: "1.0.0"   # app version — bump on app releases
maintainers:
  - name: DevOps Team
    email: devops@example.com
```

---

## 3. values.yaml — Production Defaults

```yaml
# -- Number of pod replicas
replicaCount: 2

image:
  # -- Container image repository
  repository: ghcr.io/org/my-app
  # -- Image tag (overridden by CI with Git SHA)
  tag: ""
  # -- Pull policy
  pullPolicy: IfNotPresent

# -- Image pull secrets for private registries
imagePullSecrets: []

serviceAccount:
  # -- Create a dedicated service account
  create: true
  # -- Annotations (e.g. for IRSA / Workload Identity)
  annotations: {}
  name: ""

# -- Pod security context
podSecurityContext:
  runAsNonRoot: true
  fsGroup: 1000

# -- Container security context
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: my-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

# -- Liveness probe configuration
livenessProbe:
  httpGet:
    path: /healthz
    port: http
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 3

# -- Readiness probe configuration
readinessProbe:
  httpGet:
    path: /readyz
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5

# -- Startup probe for slow-starting apps
startupProbe:
  httpGet:
    path: /healthz
    port: http
  failureThreshold: 30
  periodSeconds: 2

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

# -- Graceful shutdown settings
terminationGracePeriodSeconds: 30

# -- Node selector
nodeSelector: {}

# -- Tolerations
tolerations: []

# -- Affinity rules (prefer spread across AZs)
affinity: {}

# -- Environment variables from ConfigMap/Secrets
env: []
envFrom: []
```

---

## 4. Deployment Template

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          startupProbe:
            {{- toYaml .Values.startupProbe | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            {{- toYaml .Values.env | nindent 12 }}
          envFrom:
            {{- toYaml .Values.envFrom | nindent 12 }}
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

---

## 5. Network Policy Template

```yaml
# templates/networkpolicy.yaml
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
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - port: {{ .Values.service.targetPort }}
  egress:
    - to: []   # DNS
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    # Add specific egress rules per downstream dependency
{{- end }}
```

---

## 6. _helpers.tpl

```yaml
# templates/_helpers.tpl
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "my-app.fullname" -}}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "my-app.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
{{ include "my-app.selectorLabels" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}

{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "my-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

---

## 7. Helm Commands Cheat Sheet

```bash
# Lint chart
helm lint ./helm/my-app --strict

# Render templates locally
helm template my-app ./helm/my-app -f values-prod.yaml

# Dry-run against cluster
helm upgrade --install my-app ./helm/my-app \
  --namespace my-app-prod --dry-run

# Deploy with atomic rollback
helm upgrade --install my-app ./helm/my-app \
  --namespace my-app-prod \
  --atomic --timeout 5m \
  --set image.tag=${GIT_SHA} \
  -f values-prod.yaml

# Rollback to previous release
helm rollback my-app 0 --namespace my-app-prod

# View release history
helm history my-app --namespace my-app-prod
```

---

## 8. Anti-Patterns to Avoid

| ❌ Anti-Pattern | ✅ Correct Approach |
|----------------|---------------------|
| Deploying to `default` namespace | Create dedicated namespace per service |
| `latest` image tag | Git SHA or semver tag |
| No resource limits | Always set requests + limits |
| No probes | liveness + readiness + startup |
| `ClusterRole` for app workloads | Namespaced `Role` |
| Secrets in `values.yaml` | `existingSecret` + ExternalSecrets |
| `kubectl apply` in CI | `helm upgrade --install --atomic` |
| Single replica in prod | `minReplicas: 2` + PDB |
