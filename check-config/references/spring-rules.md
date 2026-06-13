# Spring Boot Configuration Rules — Detailed Reference

> Loaded when the skill needs deeper Spring Boot rules or the user asks for detail.

## application.yml structure

### Default (correct) setup

```yaml
# application.yml — shared baseline only
server:
  port: 8080

spring:
  application:
    name: my-service
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
```

### What belongs in profile files

```yaml
# application-dev.yml
spring:
  datasource:
    url: ${DB_URL:jdbc:mysql://localhost:3306/dev_db}
    username: ${DB_USERNAME:dev_user}
    password: ${DB_PASSWORD:}
  redis:
    host: ${REDIS_HOST:localhost}
    password: ${REDIS_PASSWORD:}
```

### What MUST NOT be in any YAML

- AWS / cloud credentials
- Private keys (PEM, BEGIN RSA PRIVATE KEY, etc.)
- API tokens / signing secrets
- Email / SMS service passwords

These go to environment variables, mounted secrets, or a secret manager (Vault, AWS Secrets Manager, etc.).

## pom.xml conventions

### Dependency version management

**Correct** — versions in `<dependencyManagement>` or BOM:
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>33.0.0-jre</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <!-- no version here — inherited from dependencyManagement -->
    </dependency>
</dependencies>
```

**Wrong** — scattered versions:
```xml
<dependencies>
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>33.0.0-jre</version>  <!-- ❌ version in leaf dependency -->
    </dependency>
</dependencies>
```

### Common unused starters to watch for

- `spring-boot-starter-webflux` when only `spring-boot-starter-web` is used
- `spring-boot-starter-data-jpa` + `spring-boot-starter-data-jdbc` together when only one is needed
- `spring-boot-starter-actuator` without any configured endpoints

## CORS configuration

### Dangerous pattern (blocker):
```java
@Bean
public CorsFilter corsFilter() {
    config.addAllowedOrigin("*");       // ❌ wildcard
    config.setAllowCredentials(true);   // ❌ credentials with wildcard
}
```

### Safe pattern:
```java
config.setAllowedOrigins(List.of(
    "http://localhost:5173",            // Vite dev server
    "https://yourdomain.com"
));
config.setAllowCredentials(true);
```

## Common security misconfigurations

| Pattern | Risk | Fix |
|---------|------|-----|
| `spring.security.user.password=plaintext` in YAML | Anyone with repo access sees it | `password: ${APP_ADMIN_PASSWORD:}` |
| `security.basic.enabled=false` | Disables all auth | Remove; use proper security config |
| `endpoints.health.sensitive=false` | Exposes health details | Set to `${MANAGEMENT_SECURITY_ENABLED:true}` |

## Docker / Docker Compose configuration

### Secrets in environment blocks

**Wrong** — bare secrets in docker-compose.yml:
```yaml
services:
  db:
    environment:
      MYSQL_ROOT_PASSWORD: MySecretP@ssw0rd!   # ❌ bare password in VCS
      MYSQL_PASSWORD: secret123
```

**Correct** — env var substitution:
```yaml
services:
  db:
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}   # ✅ from .env or CI
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

**Also correct** — Docker secrets (Swarm mode):
```yaml
services:
  db:
    secrets:
      - mysql_root_password

secrets:
  mysql_root_password:
    external: true
```

### Dockerfile secrets

**Wrong** — secrets baked into image layers:
```dockerfile
ENV DB_PASSWORD=MySecretP@ssw0rd!    # ❌ visible in docker history
```

**Correct** — build-time args + runtime env:
```dockerfile
ARG DB_PASSWORD                     # only available during build
ENV DB_PASSWORD=${DB_PASSWORD}      # ❌ still persisted in image!

# Best: only pass at runtime via docker run -e / docker-compose environment
# Dockerfile has no ENV for secrets at all
```

### Port exposure

| Pattern | Risk | Recommendation |
|---------|------|----------------|
| `ports: "3306:3306"` (binds to 0.0.0.0) | Database exposed to all network interfaces | Use `127.0.0.1:3306:3306` unless external access is intentional |
| `ports: "6379:6379"` | Redis exposed | Same — bind to loopback unless needed |
| Missing `expose:` vs `ports:` distinction | `ports:` publishes externally; `expose:` is container-to-container only | Use `expose:` for internal services |
