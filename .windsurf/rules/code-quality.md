---
trigger: always
description: Code quality guardrails — linting, Dockerfiles, Helm, YAML, dependency scanning
---

# ✅ Code Quality Rules

These rules enforce consistent, high-quality code across the DevOps stack.

---

## 1. Dockerfile Best Practices

- **Lint with Hadolint** — fail CI on `warning` level and above.
- **One concern per image** — never bundle multiple services in one container.
- **Order layers by change frequency** — OS deps → app deps → app code (maximise cache).
- **Use `.dockerignore`** — exclude `.git`, `node_modules`, `target/`, test files, docs.
- **Specify `WORKDIR`** — always set a working directory.
- **Use `COPY` over `ADD`** unless extracting archives.
- **No `apt-get upgrade`** — pin package versions instead.

```dockerfile
# ✅ Node.js multi-stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
RUN npm run build

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER nonroot
EXPOSE 3000
CMD ["dist/index.js"]
```

```dockerfile
# ✅ Spring Boot multi-stage
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -B
COPY src src
RUN ./mvnw package -DskipTests -B

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder /app/target/*.jar app.jar
USER app
EXPOSE 8080
HEALTHCHECK CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 2. Helm Chart Quality

- **`helm lint`** must pass with zero warnings before merge.
- **`helm template` + `kubectl apply --dry-run=client`** — validate rendered output.
- **Use `_helpers.tpl`** for repeated labels, selectors, and names.
- **`Chart.yaml`** — always set `appVersion`, `version`, and `description`.
- **`values.yaml`** — document every key with inline comments; group by concern.
- **Schema validation** — provide `values.schema.json` for type-safe values.
- **No hardcoded values in templates** — everything via `values.yaml`.

```yaml
# ✅ Well-documented values.yaml
# -- Number of pod replicas
replicaCount: 2

image:
  # -- Container image repository
  repository: ghcr.io/org/api
  # -- Image tag (overridden by CI)
  tag: "latest"
  # -- Image pull policy
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

---

## 3. YAML & Manifest Validation

- **`yamllint`** with strict config on all YAML files.
- **`kubeconform`** or **`kubeval`** for K8s manifest validation in CI.
- **`kube-score`** for manifest best-practice checks.
- **2-space indentation**, no tabs, no trailing whitespace.

```yaml
# .yamllint.yml
extends: default
rules:
  line-length:
    max: 120
  indentation:
    spaces: 2
  truthy:
    check-keys: false
```

---

## 4. Node.js Quality

- **ESLint** — `eslint:recommended` + `plugin:security/recommended`.
- **Prettier** — format on save; config committed to repo.
- **`npm ci`** over `npm install` — deterministic builds from lockfile.
- **`npm audit`** in CI — fail on `high` severity.
- **TypeScript preferred** — strict mode (`"strict": true`).
- **Test coverage ≥ 80 %** on new code (Jest / Vitest).
- **No `console.log`** in production — use structured logger (pino, winston).

```json
{
  "extends": [
    "eslint:recommended",
    "plugin:security/recommended",
    "plugin:@typescript-eslint/recommended"
  ],
  "rules": {
    "no-console": "error",
    "security/detect-object-injection": "warn",
    "security/detect-non-literal-fs-filename": "warn"
  }
}
```

---

## 5. Spring Boot Quality

- **Checkstyle / SpotBugs** enforced in Maven/Gradle build.
- **JaCoCo ≥ 80 %** line coverage on new code.
- **`./mvnw verify`** must pass before merge.
- **Constructor injection** — avoid `@Autowired` on fields.
- **Profile-specific configs** — `application-{env}.yml`; no secrets in any profile.
- **Structured logging** — SLF4J + Logback JSON for production.
- **API versioning** — `/api/v1/...` on all REST endpoints.

```xml
<!-- JaCoCo enforcement -->
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <executions>
    <execution>
      <goals><goal>check</goal></goals>
      <configuration>
        <rules>
          <rule>
            <limits>
              <limit>
                <counter>LINE</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.80</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

---

## 6. Git & PR Hygiene

- **Conventional Commits** — `feat:`, `fix:`, `chore:`, `docs:`, `ci:`, `refactor:`.
- **Small PRs** — ≤ 400 lines changed; split larger changes.
- **PR template** — what changed, why, how to test, rollback plan.
- **No force-push to `main`** — use `--no-ff` merges for audit trail.
- **Delete branches after merge**.

---

## 7. Dependency Management

- **Lock files committed** — `package-lock.json`, pinned `pom.xml` versions.
- **Renovate / Dependabot enabled** — auto-PRs for updates.
- **Major bumps reviewed manually** — auto-merge only for patch/minor.
- **No unused dependencies** — `depcheck` for Node.js.
- **License compliance** — `license-checker` or FOSSA; reject GPL in proprietary projects.
