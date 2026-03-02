---
name: ai-dlc-operations
description: AI-driven operations — monitoring, anomaly detection, runbook-as-code, AI-proposed incident resolution
---

# AI-DLC Operations — Skill Pack

Reference guide for the **Operations phase** of the AI-Driven Development Life Cycle.
AI monitors, detects, diagnoses, and proposes — humans approve and execute.

---

## 1. Operations Phase Overview

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Monitor    │───▶│   Detect     │───▶│   Diagnose   │───▶│   Resolve    │
│  (AI-driven) │    │  (AI-driven) │    │  (AI-driven) │    │ (Human-approved)│
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
       │                                                            │
       └────────────────── Feedback Loop ──────────────────────────┘
```

- **AI continuously monitors** metrics, logs, and traces.
- **AI detects anomalies** using baseline deviation and pattern matching.
- **AI diagnoses** root cause by correlating signals across services.
- **AI proposes** remediation — human reviews and approves before execution.

---

## 2. Observability Stack Integration

### Prometheus + Grafana + Alertmanager

```yaml
# ServiceMonitor for automatic metrics scraping
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

### Key Metrics to Monitor

| Category | Metric | Alert Threshold |
|----------|--------|-----------------|
| **Availability** | Request success rate | < 99.9% over 5m |
| **Latency** | P99 response time | > 500ms over 5m |
| **Saturation** | CPU utilization | > 80% over 10m |
| **Saturation** | Memory utilization | > 85% over 5m |
| **Errors** | 5xx error rate | > 1% over 5m |
| **Throughput** | Requests/sec | Drop > 50% from baseline |

### Application Metrics

**Node.js (prom-client):**
```typescript
import { collectDefaultMetrics, Registry, Histogram, Counter } from "prom-client";

const register = new Registry();
collectDefaultMetrics({ register });

export const httpDuration = new Histogram({
  name: "http_request_duration_seconds",
  help: "Duration of HTTP requests",
  labelNames: ["method", "route", "status"],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [register],
});

export const httpErrors = new Counter({
  name: "http_errors_total",
  help: "Total HTTP errors",
  labelNames: ["method", "route", "status"],
  registers: [register],
});
```

**Spring Boot (Micrometer + Prometheus):**
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: my-app
      environment: ${SPRING_PROFILES_ACTIVE:dev}
```

---

## 3. Runbook-as-Code

Runbooks must be **machine-readable** so the AI can propose context-aware remediations.

### Runbook Format

```yaml
# .ai-dlc/operations/runbooks/high-memory.yaml
runbook:
  id: RB-001
  name: High Memory Usage
  trigger:
    metric: container_memory_working_set_bytes
    condition: "> 85% of limit"
    duration: 5m

  diagnosis:
    steps:
      - name: Check pod memory usage
        command: kubectl top pod -l app={{ app }} -n {{ namespace }}
      - name: Check for memory leaks
        command: kubectl logs -l app={{ app }} -n {{ namespace }} --tail=200 | grep -i "heap\|memory\|oom"
      - name: Check recent deployments
        command: helm history {{ release }} -n {{ namespace }} --max 5

  resolution:
    options:
      - id: scale-vertical
        description: "Increase memory limit"
        risk: LOW
        action: |
          helm upgrade {{ release }} ./helm/{{ app }} \
            --set resources.limits.memory={{ new_limit }} \
            --namespace {{ namespace }} --atomic
      - id: restart-pods
        description: "Rolling restart to clear accumulated memory"
        risk: LOW
        action: |
          kubectl rollout restart deployment/{{ app }} -n {{ namespace }}
      - id: rollback
        description: "Rollback to previous version if recently deployed"
        risk: MEDIUM
        action: |
          helm rollback {{ release }} 0 -n {{ namespace }}

  escalation:
    if_unresolved_after: 15m
    notify: ["#oncall-channel", "pager"]
```

### Runbook Library — Common Scenarios

| ID | Scenario | Primary Resolution |
|----|----------|-------------------|
| RB-001 | High memory usage | Scale vertically or restart |
| RB-002 | High CPU usage | Scale horizontally (HPA) |
| RB-003 | 5xx error spike | Check logs → rollback if recent deploy |
| RB-004 | Latency increase | Check downstream deps → scale |
| RB-005 | Pod CrashLoopBackOff | Check logs → fix config → redeploy |
| RB-006 | Certificate expiry | Renew cert → restart ingress |
| RB-007 | Disk pressure | Clean up old images → expand PV |
| RB-008 | Database connection pool exhausted | Scale pool → check for leaks |

---

## 4. AI-Driven Anomaly Detection Patterns

### Baseline Deviation Detection

```yaml
# Prometheus alerting rule — dynamic baseline
groups:
  - name: ai-dlc-anomaly
    rules:
      - alert: LatencyAnomaly
        expr: |
          http_request_duration_seconds:p99:rate5m
            > 2 * http_request_duration_seconds:p99:rate1h_avg
        for: 5m
        annotations:
          summary: "P99 latency is 2x above 1h average"
          runbook: RB-004

      - alert: ErrorRateAnomaly
        expr: |
          rate(http_errors_total[5m])
            / rate(http_requests_total[5m]) > 0.01
        for: 5m
        annotations:
          summary: "Error rate exceeds 1%"
          runbook: RB-003

      - alert: TrafficDrop
        expr: |
          rate(http_requests_total[5m])
            < 0.5 * rate(http_requests_total[1h])
        for: 10m
        annotations:
          summary: "Traffic dropped > 50% from 1h baseline"
          runbook: RB-003
```

### Correlation Pattern

When multiple alerts fire, the AI should **correlate signals**:

```markdown
## AI Diagnosis Template

### Signals Detected
- [ ] High latency on service A (P99 > 500ms)
- [ ] Error rate spike on service B (5xx > 2%)
- [ ] Database CPU at 95%

### Correlation Analysis
1. Service B depends on Service A (downstream)
2. Service A queries the shared database
3. Database is saturated → causing Service A latency → causing Service B errors

### Proposed Root Cause
Database saturation is the root cause, cascading through the dependency chain.

### Proposed Resolution (pending human approval)
1. **Immediate**: Scale database read replicas
2. **Short-term**: Optimize slow queries identified in logs
3. **Long-term**: Add caching layer between Service A and database
```

---

## 5. AI-Proposed Incident Resolution Flow

```
Alert fires
    │
    ▼
AI reads alert metadata + linked runbook
    │
    ▼
AI runs diagnosis steps (read-only commands)
    │
    ▼
AI correlates with recent changes (deploys, config updates)
    │
    ▼
AI proposes resolution with risk assessment
    │
    ▼
┌─────────────────────────────────────┐
│  HUMAN REVIEW GATE                  │
│  - Review diagnosis                 │
│  - Approve / modify / reject        │
│  - Choose resolution option         │
└─────────────────────────────────────┘
    │
    ▼
AI executes approved resolution
    │
    ▼
AI verifies resolution (metrics returning to baseline)
    │
    ▼
AI generates post-incident context artifact
```

---

## 6. SLA Prediction & Proactive Scaling

```yaml
# HPA with custom metrics for predictive scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-predictive
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 120
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60    # proactive — scale before saturation
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

---

## 7. Post-Deployment Verification

AI must **verify every deployment** in the Operations phase:

```bash
# Automated post-deploy checks
#!/bin/bash
set -e

NAMESPACE=$1
APP=$2

echo "=== Post-Deployment Verification ==="

# 1. Rollout status
kubectl rollout status deployment/${APP} -n ${NAMESPACE} --timeout=180s

# 2. All pods running
READY=$(kubectl get pods -l app=${APP} -n ${NAMESPACE} -o jsonpath='{.items[*].status.containerStatuses[0].ready}')
echo "Pod readiness: ${READY}"

# 3. Health endpoint
POD=$(kubectl get pods -l app=${APP} -n ${NAMESPACE} -o jsonpath='{.items[0].metadata.name}')
kubectl exec ${POD} -n ${NAMESPACE} -- wget -qO- http://localhost:8080/healthz

# 4. No error spikes (wait 2 min, check error rate)
sleep 120
ERROR_RATE=$(kubectl exec ${POD} -n ${NAMESPACE} -- wget -qO- http://localhost:8080/metrics | grep http_errors_total || echo "0")
echo "Error check: ${ERROR_RATE}"

# 5. Latency within SLA
echo "=== Verification Complete ==="
```
