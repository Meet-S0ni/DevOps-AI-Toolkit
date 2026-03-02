# AI-Tools — Windsurf DevOps & AI-DLC Toolkit

A comprehensive toolkit of global rules, reusable skills, and executable workflows tailored for DevOps engineers using the **Windsurf IDE**. This repository establishes a secure, structured, and modern baseline using the **AI-Driven Development Life Cycle (AI-DLC)** methodology (inspired by AWS DevOps practices).

---

## 🏗️ Architecture

The configurations are placed under the `.windsurf/` directory in this workspace, providing contextual guidance for the IDE's AI agent to orchestrate the software development process.

```text
.windsurf/
├── rules/          # Global coding & security rules (always-on guardrails)
├── skills/         # Reusable skill packs (reference docs the AI reads on demand)
└── workflows/      # Step-by-step runbooks (slash-command style)
```

---

## 🤖 AI-Driven Development Life Cycle (AI-DLC)

This toolkit moves beyond "AI-Assisted" coding and implements the **AI-DLC** methodology, transforming the AI into a central orchestrator that proposes, clarifies, and implements solutions under human oversight.

### Key Principles Encoded in this Toolkit:
1. **AI Proposes, Human Validates:** The AI must present execution plans, architecture decisions, and incident remediation steps for human approval before applying changes.
2. **Three-Phase Awareness:**
   - **Inception:** Decompose business intent → Units & Stories → Domain Models.
   - **Construction:** Implement logic → Apply Cloud/NFR patterns → Physical Design (IaC).
   - **Operations:** Monitor → Detect Anomalies → Diagnose → Propose Resolutions.
3. **Bolt-Based Iteration:** Development happens in rapid, closely monitored cycles called "Bolts".
4. **Persistent Context:** The AI writes its plans, Domain Models, and Architecture Decision Records (ADRs) to a dedicated `.ai-dlc/` directory in the target repos.

---

## 📂 Content Overview

### 1. 🛡️ Rules (`.windsurf/rules/`)
Global guardrails the AI enforces automatically on every prompt:
- **`ai-dlc-methodology.md`** — Core tenets of the AI-DLC (roles, phases, rituals like Mob Elaboration).
- **`devops-security.md`** — Container hardening, secret management, RBAC, supply-chain checks, and AI-assisted security review logs.
- **`blast-radius.md`** — Namespace isolation, progressive delivery, PDBs, graceful shutdown, and AI-DLC phase transition gates.
- **`code-quality.md`** — Universal linting standards for Dockerfiles, Helm charts, YAML, Node.js, and Spring Boot.

### 2. 🧠 Skills (`.windsurf/skills/`)
Domain-specific knowledge bases targeted when working with particular stacks:
- **`github-actions`** — CI/CD template flows, reusable workflows, OIDC authentication, and phase-aware pipeline patterns.
- **`kubernetes-helm`** — "Domain-First to IaC" patterns, production-grade chart structures, service meshes, and network policies.
- **`trivy-sonarqube`** — Continuous scanning (FS, image, IaC), SonarQube quality gates, and automated security metrics.
- **`nodejs-springboot`** — Multi-stage docker builds, 12-factor compliance, health probes, structured logging, and graceful shutdown patterns.
- **`ai-dlc-operations`** — Machine-readable runbooks, anomaly correlation, AI diagnosis workflows, and SLA prediction.

### 3. ⚡ Workflows (`.windsurf/workflows/`)
Step-by-step executable runbooks for end-to-end automation:
- **`ai-dlc-inception.md`** — Translating a business intent into validated sub-units, stories, and domain models.
- **`ai-dlc-operations.md`** — A complete guide to AI-driven incident resolution (Detect → Diagnose → Resolve).
- **`ci-pipeline.md`** — Scaffold a standard lint → test → scan → publish GitHub Actions pipeline.
- **`cd-deploy.md`** — Scaffold a staging → production Helm deployments with OIDC and Canary rollouts.
- **`helm-chart-create.md`** — Start-to-finish production Helm chart generation, validation, and OCI publishing.
- **`security-scan.md`** — Trivy and SonarQube local/CI analysis flow.
- **`incident-response.md`** — Human-in-the-loop Kubernetes triage, rollback, fix-forward, and post-mortem procedures.

---

## 🚀 Getting Started

To use these rules in a new or existing repository in Windsurf:
1. Clone or copy the `.windsurf` folder from this toolkit into your target repository's root.
2. Ensure the IDE's rules functionality is enabled.
3. Instruct Windsurf via chat (e.g., *"Follow the `/ci-pipeline` workflow to setup GitHub Actions for this Node app"* or *"Act according to the rules in `.windsurf/rules/ai-dlc-methodology.md` to begin the Inception Phase for a new Payments Service"*).

---

## 📄 License

This repository is licensed under the **Creative Commons Attribution-NonCommercial 4.0 International (CC BY-NC 4.0)** license.

**What this means:**
- **✅ Use & Adapt:** You are free to use, copy, and modify these tools for your own personal or internal business projects.
- **❌ No Commercial Sale:** You may **not** sell this toolkit or use it for primary commercial advantage or monetary compensation.
- **ℹ️ Attribution:** You must give appropriate credit if you share or redistribute these tools.

Refer to the [LICENSE](LICENSE) file for the full legal text.
