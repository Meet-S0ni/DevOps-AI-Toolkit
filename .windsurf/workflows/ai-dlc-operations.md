---
description: AI-DLC Operations phase — AI-driven deployment verification, anomaly diagnosis, human-approved remediation
---

# AI-DLC Operations Phase Workflow

Step-by-step workflow for the **Operations phase** — AI monitors, detects, diagnoses,
and proposes resolutions. Humans review and approve before execution.

---

## Overview

```
Deploy (from Construction phase)
      │
      ▼
AI verifies deployment health
      │
      ▼
AI monitors continuously
      │
      ├── Anomaly detected ──▶ AI diagnoses ──▶ AI proposes fix ──▶ Human approves
      │                                                                    │
      │                                                              AI executes
      │                                                                    │
      └────────────────────── Feedback Loop ◀─────────────────────────────┘
```

---

## Steps

### 1. Post-deployment verification

After every deployment, AI runs automated verification:

```bash
# Verify deployment rolled out successfully
kubectl rollout status deployment/<app> -n <namespace> --timeout=180s

# Confirm all pods are ready
kubectl get pods -l app=<app> -n <namespace>

# Hit health endpoints
kubectl exec <pod> -n <namespace> -- wget -qO- http://localhost:8080/healthz
kubectl exec <pod> -n <namespace> -- wget -qO- http://localhost:8080/readyz

# Check for error spikes (wait 2 minutes for traffic to flow)
sleep 120

# Compare current error rate to pre-deployment baseline
# If error rate increased > 0.5% → flag for human review
```

**AI Decision Point:**
- ✅ All checks pass → log success to `.ai-dlc/bolts/bolt-XXX.md`
- ⚠️ Health check fails → propose rollback to human
- ❌ Error rate spike → trigger automatic rollback (if `--atomic` was used)

### 2. Set up monitoring context

AI should understand the service's monitoring baseline.

```markdown
<!-- .ai-dlc/operations/monitoring-context.md -->
# Monitoring Context: [Service Name]

## Key Metrics
| Metric | Normal Range | Alert Threshold |
|--------|-------------|-----------------|
| P99 latency | 50-200ms | > 500ms |
| Error rate | 0-0.1% | > 1% |
| CPU usage | 20-60% | > 80% |
| Memory usage | 40-70% | > 85% |
| Request rate | 100-500 rps | Drop > 50% |

## Dependencies
- Database: [connection details, normal query time]
- Cache: [Redis, hit rate baseline]
- Downstream APIs: [services, expected latency]

## Recent Changes
- [Date]: [deployment/config change description]
```

### 3. AI-driven anomaly detection

When alerts fire, AI follows a structured diagnosis process:

```markdown
## AI Diagnosis Procedure

### Step 1: Identify the alert
- Which metric is anomalous?
- What is its current value vs. expected baseline?
- How long has it been anomalous?

### Step 2: Check for recent changes
- Was there a deployment in the last 2 hours?
- Were there config changes?
- Were there infrastructure changes?

### Step 3: Correlate with other signals
- Are other services affected?
- Is the database healthy?
- Are there network issues?

### Step 4: Identify root cause
- Is this a code issue (recent deploy)?
- Is this a resource issue (capacity)?
- Is this an external dependency issue?

### Step 5: Propose resolution
- Present root cause analysis
- List resolution options with risk levels
- Recommend one option
- **Wait for human approval**
```

### 4. AI-proposed resolution template

When the AI has a diagnosis, it presents this to the human:

```markdown
## 🚨 Incident Analysis

**Alert:** [Alert name and description]
**Severity:** SEV-[1-4]
**Duration:** [How long the issue has persisted]

### Diagnosis
[Root cause analysis — what the AI found]

### Evidence
- Metric A: [current value] (baseline: [normal value])
- Log pattern: [relevant log excerpt]
- Recent change: [deployment or config change if any]

### Resolution Options

| # | Option | Risk | Expected Recovery |
|---|--------|------|-------------------|
| 1 | [Preferred option] | LOW | 2-5 min |
| 2 | [Alternative option] | MEDIUM | 5-10 min |
| 3 | Full rollback | LOW | 3-5 min |

### Recommendation
I recommend **Option 1** because [reasoning].

### ⏸️ Awaiting your approval to proceed.
```

### 5. Execute approved resolution

After human approval:

```bash
# AI executes the approved resolution
# Example: Rolling restart
kubectl rollout restart deployment/<app> -n <namespace>

# Monitor recovery
kubectl rollout status deployment/<app> -n <namespace> --timeout=120s

# Verify metrics returning to baseline (wait 3 minutes)
sleep 180

# Confirm error rate back to normal
echo "Post-resolution error rate: [check metrics]"
```

### 6. Generate post-incident context

After resolution, AI persists the incident for future learning:

```markdown
<!-- .ai-dlc/operations/incidents/INC-YYYY-MM-DD-001.md -->
# Incident: [Title]

## Timeline
- HH:MM — Alert fired: [description]
- HH:MM — AI diagnosis completed
- HH:MM — Human approved resolution
- HH:MM — Resolution executed
- HH:MM — Service verified healthy

## Root Cause
[Technical root cause]

## Resolution Applied
[What was done to fix it]

## Prevention
- [ ] [Action to prevent recurrence]
- [ ] [Monitoring improvement]
- [ ] [Runbook update]

## Context for AI
[What the AI should remember for similar future incidents]
```

### 7. Continuous improvement loop

AI uses accumulated operations context to **improve over time**:

- **Runbook refinement** — if a resolution worked, strengthen the runbook; if not, update it.
- **Alert tuning** — if alerts are too noisy (false positives), propose threshold adjustments.
- **Pattern recognition** — if similar incidents recur, propose systemic fixes.
- **Capacity planning** — based on traffic trends, propose proactive scaling.

```markdown
## Monthly AI Operations Review

### Alert Quality
- Total alerts: [N]
- True positives: [N] ([%])
- False positives: [N] ([%])
- Proposed threshold changes: [list]

### Incident Patterns
- [Pattern 1]: occurred [N] times → proposed fix: [description]
- [Pattern 2]: occurred [N] times → proposed fix: [description]

### Capacity Trends
- [Service A]: traffic growing [X]% monthly → scale by [date]
- [Service B]: stable, no action needed
```

---

## Checklist

- [ ] Post-deployment verification automated
- [ ] Monitoring context documented per service
- [ ] Runbooks created for top failure scenarios
- [ ] Alert rules configured with appropriate thresholds
- [ ] AI diagnosis procedure followed for every incident
- [ ] Human approval gate enforced before production changes
- [ ] Post-incident artifacts saved to `.ai-dlc/operations/incidents/`
- [ ] Monthly operations review conducted
