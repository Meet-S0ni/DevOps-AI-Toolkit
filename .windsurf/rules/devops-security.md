---
trigger: always
description: Security best practices for DevOps — containers, secrets, RBAC, supply chain, network policies
---

# 🔒 DevOps Security Rules

These rules are **always active**. Every suggestion the AI makes MUST comply with them.

---

## 1. Container Image Hardening

- **Use minimal base images** — prefer `distroless`, `alpine`, or `scratch`. Never use `latest` tag.
- **Pin image digests** in production (`image: node:20-alpine@sha256:...`).
- **Run as non-root** — every Dockerfile must include:
  ```dockerfile
  RUN addgroup -S app && adduser -S app -G app
  USER app
  ```
- **Multi-stage builds** — build in one stage, copy artifacts to a clean runtime stage.
- **No secrets in images** — never `COPY .env`, never `ARG SECRET`. Use runtime injection only.
- **Minimize layers** — combine `RUN` commands with `&&`; clean caches in the same layer.
- **Set `HEALTHCHECK`** in every Dockerfile.

---

## 2. Secret Management

- **Never hardcode secrets** — no tokens, passwords, or keys in source code, Helm values, or CI files.
- **Use external secret stores** — prefer Kubernetes `ExternalSecret` (ESO) backed by Vault / AWS SSM / GCP Secret Manager.
- **GitHub Actions secrets** — reference via `${{ secrets.NAME }}`; never echo or log them.
- **Helm** — use `existingSecret` pattern; never put sensitive values in `values.yaml` committed to git.
- **Rotate secrets regularly** — document rotation procedure alongside every secret.
- **`.gitignore`** — ensure `*.env`, `*.pem`, `*.key`, `kubeconfig` are ignored.

---

## 3. Least-Privilege RBAC

- **Kubernetes** — create dedicated `ServiceAccount` per workload; never use `default`.
- **Bind `Role` (namespaced), not `ClusterRole`**, unless cross-namespace access is genuinely required.
- **No wildcard verbs** — avoid `verbs: ["*"]` or `resources: ["*"]`.
- **GitHub Actions** — use `permissions:` block to scope token to minimum (e.g., `contents: read`).
- **OIDC over long-lived tokens** — use GitHub OIDC for cloud provider auth instead of static credentials.

```yaml
# ✅ Good — scoped Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: payments
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
```

```yaml
# ❌ Bad — overly permissive ClusterRole
kind: ClusterRole
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

---

## 4. Network Policies

- **Default-deny ingress AND egress** per namespace:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: default-deny-all
    namespace: {{ .Values.namespace }}
  spec:
    podSelector: {}
    policyTypes: [Ingress, Egress]
  ```
- **Explicitly allow only required traffic** between services.
- **Restrict egress to known CIDRs** — block arbitrary outbound internet access from workloads.

---

## 5. Supply-Chain Security

### Trivy
- **Scan every image before push** — fail CI on `CRITICAL` or `HIGH` vulnerabilities.
- **Scan IaC** (Helm charts, K8s manifests, Terraform) with `trivy config`.
- **Scan lockfiles** (`package-lock.json` / `pom.xml`) with `trivy fs` for dependency CVEs.

### SonarQube
- **Quality Gate must pass** before merge — enforce via CI status check.
- **Zero new critical / blocker issues** policy.
- **Minimum 80 % code coverage** on new code.
- **Enable secret-detection rules** in SonarQube.

### General
- **Sign container images** with Cosign / Sigstore.
- **Enable Dependabot / Renovate** for automated dependency updates.
- **Pin all GitHub Actions to commit SHA**, not tags:
  ```yaml
  # ✅ Good
  uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29  # v4.1.6
  # ❌ Bad
  uses: actions/checkout@v4
  ```

---

## 6. Kubernetes Pod Security

- **Set `securityContext`** on every container:
  ```yaml
  securityContext:
    runAsNonRoot: true
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    capabilities:
      drop: ["ALL"]
  ```
- **Enforce Pod Security Standards** — label namespaces with `pod-security.kubernetes.io/enforce: restricted`.
- **Set resource requests AND limits** — prevent noisy-neighbour and OOM issues.

---

## 7. CI/CD Pipeline Security

- **Branch protection** — require PR reviews, status checks, and signed commits on `main`.
- **Immutable artifacts** — tag images with Git SHA; never overwrite existing tags.
- **Separate build and deploy credentials** — build pipeline must not hold production deploy access.
- **Audit logging** — enable GitHub audit log streaming and Kubernetes audit logs.

---

## 8. Spring Boot & Node.js Specifics

### Spring Boot
- Disable actuator endpoints in production or secure them behind authentication.
- Use `application-prod.yml` profile — never expose debug/dev settings in production.
- Enable CSRF protection, CORS whitelisting, and rate-limiting.

### Node.js
- Use `helmet` middleware for HTTP header hardening.
- Run `npm audit` in CI — fail on `high` severity.
- Avoid `eval()` and `child_process.exec()` with user input — flag as security violation.
- Set `NODE_ENV=production` in container images.

---

## 9. AI-Assisted Security Review (AI-DLC)

Following the AI-DLC methodology, the AI must act as a **security collaborator** — proposing fixes
while deferring critical security decisions to humans.

### AI Security Workflow
1. **Scan** — AI runs Trivy / SonarQube and interprets results.
2. **Triage** — AI categorizes findings by severity and exploitability.
3. **Propose** — AI suggests a fix with risk assessment for each finding.
4. **Wait** — Human reviews and approves before any security-related change is applied.
5. **Log** — AI records the security decision in an ADR.

### Rules
- **Never auto-fix security vulnerabilities** — every remediation requires human approval.
- **Maintain a security decisions log** at `.ai-dlc/architecture/decisions/` with `security-` prefix.
- **AI must explain the vulnerability** in plain language — what it is, how it's exploitable, what the fix does.
- **Prioritize by blast radius** — CRITICAL + publicly exposed > CRITICAL + internal > HIGH.
- **Track accepted risks** with expiration dates — revisit quarterly.

```markdown
<!-- Example: .ai-dlc/architecture/decisions/security-adr-001.md -->
# Security ADR-001: CVE-2024-XXXXX in base image

## Severity: CRITICAL
## Status: Remediated

## Vulnerability
Remote code execution via crafted HTTP header in Node.js < 20.11.1.

## Decision
Update base image from node:20.10-alpine to node:20.11.1-alpine.

## Risk if Not Fixed
Attackable from public internet; affects all endpoints.

## Verification
- Trivy re-scan shows CVE resolved.
- All tests pass on updated base image.
```
