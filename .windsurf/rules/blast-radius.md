---
trigger: always
description: Blast radius reduction — isolation, progressive delivery, rollback, resource boundaries
---

# 💥 Reducing Blast Radius — Rules

These rules ensure every change is **scoped, reversible, and isolated**.

---

## 1. Namespace Isolation

- **One service (or tightly coupled group) per namespace**.
- **Every namespace MUST have**:
  - `ResourceQuota` (CPU, memory, pods, services)
  - `LimitRange` (default request/limit per container)
  - `NetworkPolicy` (default-deny, explicit allow)
- **Naming convention**: `<team>-<service>-<env>` (e.g. `payments-api-prod`).
- **Never deploy to the `default` namespace**.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: payments-api-prod
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

---

## 2. Progressive Delivery

- **Canary deployments** — roll out to ≤ 10 % of traffic first; validate error rate and latency before proceeding.
- **Blue / Green** — for database-migration-coupled releases; keep old version warm until validation passes.
- **Feature flags** — decouple deployment from release; prefer runtime toggles over branching.
- **Never deploy directly to 100 % traffic** — always use a staged rollout strategy.

### Helm Rollout Strategy
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # add 1 pod at a time
    maxUnavailable: 0    # never reduce below desired count
```

---

## 3. Pod Disruption Budgets (PDB)

- **Every production Deployment with ≥ 2 replicas MUST have a PDB**.
- **`minAvailable` ≥ 50 %** or `maxUnavailable: 1`.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: "50%"
  selector:
    matchLabels:
      app: api
```

---

## 4. Resource Requests & Limits

- **Requests = observed P95 usage**; **Limits = 2× requests** (tuned per workload).
- **Always set both CPU and memory** — missing limits → unbound consumption.
- **Use VPA in recommend mode** to right-size over time.

```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

---

## 5. Rollback Strategy

- **Helm** — deploy with `--atomic --timeout 5m`; auto-rollback on failure.
- **Keep last 5 releases** (`--history-max 5`) for quick `helm rollback`.
- **Tag images with Git SHA** — rollback to exact commit is trivial.
- **Database migrations must be backward-compatible** (expand-contract pattern).
- **Never delete old resources before new ones are healthy**.

```bash
# Atomic deploy with auto-rollback
helm upgrade --install my-app ./chart \
  --namespace payments-api-prod \
  --atomic \
  --timeout 5m \
  --set image.tag=${GIT_SHA}
```

---

## 6. Health Probes

- **Every container MUST define** `livenessProbe`, `readinessProbe`, and `startupProbe` (for slow starters like Spring Boot).
- **Probes must hit a dedicated health endpoint**, not the main traffic path.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 2
```

---

## 7. Horizontal Pod Autoscaler (HPA)

- **Use HPA for stateless workloads** — scale on CPU (target 70 %) and custom metrics.
- **`minReplicas ≥ 2`** in production for high availability.
- **Pair with PDB** to prevent scale-down outages.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## 8. Change Scope Rules

- **One service per PR** — never bundle unrelated service changes.
- **Database migrations in a separate PR** from application code.
- **Infrastructure changes reviewed by ≥ 2 engineers**.
- **Environment promotion**: `dev → staging → prod` — never skip staging.
- **Terraform / IaC** — always `plan` before `apply`; require approval for `prod`.

---

## 9. Graceful Shutdown

- **Handle `SIGTERM`** in Node.js and Spring Boot — stop accepting requests, drain in-flight, close DB connections, flush logs.
- **`terminationGracePeriodSeconds` ≥ drain time** (default 30 s).
- **`preStop` hook** to let kube-proxy update endpoints:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
```
