---
name: nodejs-springboot
description: Dockerfile patterns, health endpoints, graceful shutdown, env config, 12-factor compliance for Node.js & Spring Boot
---

# Node.js & Spring Boot — Skill Pack

Reference guide for building **production-ready, 12-factor compliant** Node.js and Spring Boot applications.

---

## 1. Dockerfile — Node.js (Production)

```dockerfile
# ---------- BUILD ----------
FROM node:20-alpine AS builder
WORKDIR /app

# Install dependencies first (cache layer)
COPY package.json package-lock.json ./
RUN npm ci

# Copy source and build
COPY tsconfig.json ./
COPY src/ src/
RUN npm run build

# Remove dev dependencies
RUN npm prune --production

# ---------- RUNTIME ----------
FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

USER nonroot
EXPOSE 3000

CMD ["dist/index.js"]
```

---

## 2. Dockerfile — Spring Boot (Production)

```dockerfile
# ---------- BUILD ----------
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

# Cache Maven dependencies
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN chmod +x mvnw && ./mvnw dependency:go-offline -B

# Build application
COPY src src
RUN ./mvnw package -DskipTests -B

# Extract layered JAR for optimal Docker caching
RUN java -Djarmode=layertools -jar target/*.jar extract --destination extracted

# ---------- RUNTIME ----------
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

RUN addgroup -S app && adduser -S app -G app

# Copy layers in order of change frequency
COPY --from=builder /app/extracted/dependencies/ ./
COPY --from=builder /app/extracted/spring-boot-loader/ ./
COPY --from=builder /app/extracted/snapshot-dependencies/ ./
COPY --from=builder /app/extracted/application/ ./

USER app
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## 3. Health Endpoints — Node.js

```typescript
// src/health.ts
import { Router, Request, Response } from "express";

const router = Router();
let isReady = false;

// Call this after all dependencies are initialized
export function markReady(): void {
  isReady = true;
}

// Liveness — is the process alive?
router.get("/healthz", (_req: Request, res: Response) => {
  res.status(200).json({ status: "UP" });
});

// Readiness — can it serve traffic?
router.get("/readyz", (_req: Request, res: Response) => {
  if (!isReady) {
    return res.status(503).json({ status: "NOT_READY" });
  }
  // Add downstream checks (DB, cache, etc.)
  res.status(200).json({ status: "READY" });
});

export default router;
```

---

## 4. Health Endpoints — Spring Boot

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true   # enables /actuator/health/liveness & /readiness
  health:
    db:
      enabled: true
    diskSpace:
      enabled: true
```

```java
// Custom health indicator
@Component
public class DownstreamHealthIndicator implements HealthIndicator {
    
    private final RestTemplate restTemplate;

    public DownstreamHealthIndicator(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @Override
    public Health health() {
        try {
            restTemplate.getForEntity("http://downstream-service/healthz", String.class);
            return Health.up().build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

---

## 5. Graceful Shutdown — Node.js

```typescript
// src/shutdown.ts
import { Server } from "http";
import { Pool } from "pg"; // or your DB client

export function setupGracefulShutdown(server: Server, dbPool: Pool): void {
  const shutdown = async (signal: string) => {
    console.log(`Received ${signal}. Starting graceful shutdown...`);

    // 1. Stop accepting new connections
    server.close(() => {
      console.log("HTTP server closed.");
    });

    // 2. Wait for in-flight requests (timeout after 25s)
    const timeout = setTimeout(() => {
      console.error("Graceful shutdown timed out. Forcing exit.");
      process.exit(1);
    }, 25_000);

    try {
      // 3. Close database connections
      await dbPool.end();
      console.log("Database pool closed.");

      clearTimeout(timeout);
      process.exit(0);
    } catch (err) {
      console.error("Error during shutdown:", err);
      process.exit(1);
    }
  };

  process.on("SIGTERM", () => shutdown("SIGTERM"));
  process.on("SIGINT", () => shutdown("SIGINT"));
}
```

---

## 6. Graceful Shutdown — Spring Boot

```yaml
# application.yml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
```

```java
// @PreDestroy for custom cleanup
@Component
public class CleanupHandler {

    private static final Logger log = LoggerFactory.getLogger(CleanupHandler.class);

    @PreDestroy
    public void onShutdown() {
        log.info("Running cleanup tasks before shutdown...");
        // Flush caches, close connections, etc.
    }
}
```

---

## 7. Environment Configuration — 12-Factor

### Node.js
```typescript
// src/config.ts — validated env config
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "staging", "production"]),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().optional(),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

export const config = envSchema.parse(process.env);
```

### Spring Boot
```yaml
# application.yml — externalized config
server:
  port: ${PORT:8080}

spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: ${DB_POOL_SIZE:10}
      connection-timeout: 5000

logging:
  level:
    root: ${LOG_LEVEL:INFO}
  pattern:
    console: '{"timestamp":"%d","level":"%p","logger":"%logger","message":"%m"}%n'
```

---

## 8. Structured Logging

### Node.js (pino)
```typescript
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL || "info",
  formatters: {
    level: (label) => ({ level: label }),
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  // Redact sensitive fields
  redact: ["req.headers.authorization", "req.headers.cookie"],
});
```

### Spring Boot (Logback JSON)
```xml
<!-- src/main/resources/logback-spring.xml -->
<configuration>
  <springProfile name="prod,staging">
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <fieldNames>
          <timestamp>timestamp</timestamp>
          <version>[ignore]</version>
        </fieldNames>
      </encoder>
    </appender>
    <root level="INFO">
      <appender-ref ref="JSON" />
    </root>
  </springProfile>

  <springProfile name="dev">
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
      <encoder>
        <pattern>%d{HH:mm:ss} %-5level %logger{36} - %msg%n</pattern>
      </encoder>
    </appender>
    <root level="DEBUG">
      <appender-ref ref="CONSOLE" />
    </root>
  </springProfile>
</configuration>
```

---

## 9. .dockerignore Templates

### Node.js
```
.git
.github
.vscode
node_modules
coverage
dist
*.md
*.log
.env*
Dockerfile
docker-compose*.yml
.dockerignore
```

### Spring Boot
```
.git
.github
.vscode
.idea
target
*.md
*.log
.env*
Dockerfile
docker-compose*.yml
.dockerignore
```

---

## 10. 12-Factor Compliance Checklist

| Factor | Node.js | Spring Boot |
|--------|---------|-------------|
| I. Codebase | Git repo | Git repo |
| II. Dependencies | `package-lock.json` | `pom.xml` with pinned versions |
| III. Config | `process.env` + validation | `application.yml` + `${ENV}` |
| IV. Backing Services | Connection URLs from env | DataSource from env |
| V. Build/Release/Run | `npm ci && npm build` → Docker | `mvnw package` → Docker |
| VI. Processes | Stateless; use Redis for sessions | Stateless; use Redis for sessions |
| VII. Port Binding | `PORT` env var | `server.port` from env |
| VIII. Concurrency | Scale via K8s HPA | Scale via K8s HPA |
| IX. Disposability | Graceful shutdown handlers | `server.shutdown: graceful` |
| X. Dev/Prod Parity | Docker Compose mirroring prod | Docker Compose mirroring prod |
| XI. Logs | JSON to stdout (pino) | JSON to stdout (Logback) |
| XII. Admin Processes | One-off scripts in `scripts/` | `CommandLineRunner` beans |
