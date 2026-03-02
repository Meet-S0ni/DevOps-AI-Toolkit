---
description: Rollback, debug, and recover from a failed Kubernetes deployment
---

# Incident Response Workflow

Step-by-step runbook for handling failed deployments and production incidents.

---

## Severity Levels

| Level | Description | Response Time |
|-------|-------------|---------------|
| SEV-1 | Complete outage, all users affected | Immediate |
| SEV-2 | Partial outage, degraded service | < 15 min |
| SEV-3 | Minor issue, workaround available | < 1 hour |
| SEV-4 | No user impact, internal only | Next business day |

---

## Step 1 — Detect & Assess

```bash
# Check deployment status
kubectl rollout status deployment/<app> -n <namespace>

# Check pod health
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace>

# Check recent events
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20

# Check application logs
kubectl logs -l app=<app-name> -n <namespace> --tail=100
kubectl logs <pod-name> -n <namespace> --previous   # crashed container
```

---

## Step 2 — Immediate Rollback (if needed)

### Helm Rollback
```bash
# View release history
helm history <release> -n <namespace>

# Rollback to previous release
helm rollback <release> 0 -n <namespace>

# Rollback to specific revision
helm rollback <release> <revision> -n <namespace>

# Verify rollback
kubectl rollout status deployment/<app> -n <namespace>
```

### kubectl Rollback (if not using Helm)
```bash
# View rollout history
kubectl rollout history deployment/<app> -n <namespace>

# Rollback to previous revision
kubectl rollout undo deployment/<app> -n <namespace>

# Rollback to specific revision
kubectl rollout undo deployment/<app> -n <namespace> --to-revision=<N>
```

---

## Step 3 — Diagnose Root Cause

### Common Failure Patterns

#### A. CrashLoopBackOff
```bash
# Check exit code and logs
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous

# Common causes:
# - Missing environment variables
# - Database connection failure
# - Application startup error
# - OOMKilled (check resources.limits.memory)
```

#### B. ImagePullBackOff
```bash
# Check image name and tag
kubectl describe pod <pod> -n <namespace> | grep -A5 "Image"

# Common causes:
# - Wrong image tag (typo in SHA)
# - Registry auth expired
# - Image doesn't exist
```

#### C. Readiness probe failing
```bash
# Check probe configuration
kubectl get deployment <app> -n <namespace> -o yaml | grep -A10 readinessProbe

# Test health endpoint from inside the pod
kubectl exec -it <pod> -n <namespace> -- wget -qO- http://localhost:8080/readyz

# Common causes:
# - App starting slowly (increase initialDelaySeconds)
# - Dependency not available
# - Wrong port or path
```

#### D. OOMKilled
```bash
# Check memory usage
kubectl top pod <pod> -n <namespace>

# Check if OOMKilled
kubectl describe pod <pod> -n <namespace> | grep -i oom

# Fix: increase memory limits or investigate memory leak
```

#### E. Resource quota exceeded
```bash
# Check quota usage
kubectl describe resourcequota -n <namespace>

# Fix: increase quota or scale down other workloads
```

---

## Step 4 — Communicate

### Status Update Template
```
**Incident: [Brief description]**
**Severity:** SEV-X
**Status:** Investigating / Mitigated / Resolved
**Impact:** [Who and what is affected]
**Timeline:**
- HH:MM — Issue detected
- HH:MM — Rollback initiated
- HH:MM — Service restored
**Root Cause:** [Brief description]
**Next Steps:** [What comes next]
```

---

## Step 5 — Fix Forward (after rollback)

```bash
# 1. Reproduce in staging
helm upgrade --install <app> ./helm/<app> \
  --namespace <app>-staging \
  --set image.tag=<broken-sha> \
  -f values-staging.yaml

# 2. Debug in staging
kubectl logs -f -l app=<app> -n <app>-staging
kubectl exec -it <pod> -n <app>-staging -- sh

# 3. Fix, test, push new commit

# 4. Deploy fix through normal CI/CD pipeline
# NEVER hotfix directly to production

# 5. Verify in staging first, then promote to production
```

---

## Step 6 — Post-Incident Review

### Blameless Post-Mortem Template

1. **What happened?** — Timeline of events
2. **What was the impact?** — Users affected, duration, severity
3. **Root cause** — Technical root cause analysis
4. **What went well?** — Detection, response, communication
5. **What could be improved?** — Process, tooling, monitoring
6. **Action items:**
   - [ ] Prevent recurrence (fix + test)
   - [ ] Improve detection (alerting, monitoring)
   - [ ] Improve response (runbooks, automation)
   - [ ] Update documentation

---

## Quick Reference — Emergency Commands

```bash
# Scale down to zero (circuit breaker)
kubectl scale deployment/<app> --replicas=0 -n <namespace>

# Scale back up
kubectl scale deployment/<app> --replicas=3 -n <namespace>

# Force restart all pods
kubectl rollout restart deployment/<app> -n <namespace>

# Block traffic (emergency network policy)
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: emergency-deny-all
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app: <app>
  policyTypes: [Ingress]
EOF

# Remove emergency block
kubectl delete networkpolicy emergency-deny-all -n <namespace>
```

---

## AI-Driven Incident Resolution (AI-DLC)

Following the AI-DLC Operations phase pattern, the AI acts as a **first responder** —
analyzing alerts, correlating signals, and proposing resolutions for human approval.

### AI Resolution Workflow

```
Alert fires
    │
    ▼
AI reads alert + linked runbook (.ai-dlc/operations/runbooks/)
    │
    ▼
AI runs read-only diagnosis commands
    │
    ▼
AI correlates with:
  - Recent deployments (helm history)
  - Other service alerts
  - Past incidents (.ai-dlc/operations/incidents/)
    │
    ▼
AI presents diagnosis + resolution options + risk levels
    │
    ▼
┌──────────────────────────────┐
│   🛑 HUMAN APPROVAL GATE    │
│   Review → Approve/Modify   │
└──────────────────────────────┘
    │
    ▼
AI executes approved action
    │
    ▼
AI verifies metrics returning to baseline
    │
    ▼
AI saves incident context for future learning
```

### AI Diagnosis Template

When an alert fires, the AI should produce this analysis:

```markdown
## 🚨 Incident: [Alert Name]

**Time:** [timestamp]
**Severity:** SEV-[1-4]
**Service:** [affected service]
**Alert:** [metric and threshold breached]

### Evidence
| Signal | Current | Baseline | Status |
|--------|---------|----------|--------|
| Error rate | 3.2% | 0.1% | ❌ |
| P99 latency | 1200ms | 180ms | ❌ |
| CPU usage | 45% | 40% | ✅ |
| Recent deploy | 25 min ago | — | ⚠️ |

### Root Cause Analysis
[AI's assessment based on correlating the signals above]

### Similar Past Incidents
- INC-2024-11-15: [similar pattern, resolution was X]

### Resolution Options
| # | Action | Risk | ETA |
|---|--------|------|-----|
| 1 | Rollback to prev release | LOW | 3 min |
| 2 | Scale up replicas | LOW | 2 min |
| 3 | Fix forward (hotfix) | HIGH | 30 min |

### Recommendation
Option 1 — rollback. Recent deployment correlates with symptom onset.

⏸️ **Awaiting approval to proceed.**
```

### Post-Incident: Save Context

After resolution, AI must persist the incident for future learning:

```bash
# Save to .ai-dlc/operations/incidents/
cat > .ai-dlc/operations/incidents/INC-$(date +%Y-%m-%d)-001.md << 'EOF'
# Incident: [Title]
## Root Cause: [description]
## Resolution: [what was done]
## Prevention: [what should change]
## AI Learning: [what the AI should remember for next time]
EOF
```

### Rules
- **AI must never execute production changes without human approval** — even if confidence is high.
- **AI must check past incidents** before proposing a resolution — avoid repeating ineffective fixes.
- **AI must log every incident** — the operations context grows over time, improving future diagnoses.
- **Runbooks are the AI's playbook** — keep them updated at `.ai-dlc/operations/runbooks/`.
