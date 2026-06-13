---
name: check-config
description: >
  Audit and normalize configuration files for a Spring Boot + Vue/Vite full-stack project.
  Checks for leaked secrets, missing placeholders, format inconsistencies, and violations of
  the red lines defined in CLAUDE.md. TRIGGER: "check config", "config compliance",
  "normalize config", "audit config files", "检查配置", "配置合规", "配置审计", "规范配置",
  "配置检查", "scan config".
user-invocable: true
allowed-tools:
  - Read
  - Grep
  - Glob
  - Edit
  - Bash
arguments:
  - target
argument-hint: "[backend|frontend|all，默认 all]"
---

# 🔍 Config Compliance Check — Spring Boot + Vue

You are a **project config auditor**. Follow the SOP below. Never write production values on your own — stop and list every decision that requires human input.

## Pre-flight: Read the constitution

Before anything else, **read `CLAUDE.md`** at the project root. Every rule you enforce comes from there. If CLAUDE.md doesn't exist, say so and stop — the audit has no baseline.

---

## 0. Determine scope

Normalize `$target` with fuzzy matching (case-insensitive, strip whitespace):

| Input | Resolved scope |
|-------|---------------|
| empty / `all` / `both` / `全部` / `全盘` | `all` — backend + frontend |
| `backend` / `back` / `spring` / `后端` / `java` | `backend` — Spring Boot only |
| `frontend` / `front` / `vue` / `前端` / `vite` | `frontend` — Vue/Vite only |
| anything else | Treat as a hint but default to `all`, warn user |

First, print the file paths you found so the user can verify:

- Backend: `**/src/main/resources/application*.yml`, `**/src/main/resources/application*.properties`, `pom.xml` / `build.gradle`
- Frontend: `frontend/vite.config.ts`, `frontend/tsconfig.json`, `frontend/.env*`, `frontend/package.json`
- Docker (if present): `docker-compose.yml`, `docker-compose.*.yml`, `Dockerfile`

---

## 1. Spring Boot config audit

### 1.1 Plaintext secret scan (highest priority — ❌ BLOCKER)

Scan `application*.yml` / `application*.properties`:

| Rule | Verdict |
|---|---|
| `password:` / `secret:` / `token:` / `api-key:` / `access-key:` followed by a bare string (not `${...}` placeholder) | ❌ Must use `${ENV_VAR}` |
| Content that looks like a private key / certificate (`BEGIN PRIVATE KEY`, `BEGIN RSA PRIVATE KEY`, etc.) | ❌ Must move to a mounted volume or secret manager |
| `spring.datasource.url` containing `username:password@` (inline credentials) | ❌ Must split into env vars |
| `spring.security.oauth2.client.registration.*.client-secret` with a bare value | ❌ Must use `${OAUTH_CLIENT_SECRET}` |
| `jwt.secret` / `jwt.secret-key` / `app.jwt-secret` with a bare string | ❌ Must use `${JWT_SECRET}` |

**Also scan Java configuration files** (`**/*Config.java`, `**/application/*.java`):

| Rule | Verdict |
|---|---|
| `@Value("${...}")` with a hardcoded default that IS a secret (e.g., `@Value("${db.password:myRealPassword}")`) | ❌ The default value after `:` is a real secret — remove it or use a dummy placeholder |
| `@Value` annotating a field but the YAML/properties key is a bare string (not a placeholder) | ❌ Trace back to the config file and fix there |

For each hit, output:

```
❌ [BLOCKER] backend/src/main/resources/application-dev.yml:6
   password: mysecret123   ← bare password
   Fix: password: ${DB_PASSWORD:__MUST_SET_IN_ENV__}
```

### 1.2 Profile & format structure check

- `application.yml` must NOT contain profile-specific values (DB URLs, third-party keys, etc.)
  - If it does → `⚠️ WARNING`, suggest moving to `application-{profile}.yml`
- Check whether the project mixes `application.yml` AND `application.properties` in the same `src/main/resources/`
  - Both present → `⚠️ WARNING: Mixed YAML and .properties — pick one and consolidate to avoid precedence confusion`. Spring Boot loads YAML first, then `.properties` overrides it, which causes hard-to-debug surprises.
- If `application.properties` is used in a YAML-first project (or vice versa), flag it

### 1.3 Spring best practices

- `management.endpoints.web.exposure.include=*` allowed only in non-prod; prod must use a whitelist → WARNING
- `spring.profiles.active` must use env injection (`${SPRING_PROFILES_ACTIVE:dev}`), not a hardcoded value → WARNING if not
- CORS: `allowedOrigins: "*"` combined with `allowCredentials: true` → ❌ BLOCKER
- **pom.xml**: scan for unused `spring-boot-starter-*` dependencies (e.g., `webflux` alongside `web`, `data-jpa` + `data-jdbc` together, `actuator` with no configured endpoints). Flag them → WARNING with removal suggestion.

> 📖 For detailed examples of CORS patterns, pom.xml version management, and security config pitfalls, read `references/spring-rules.md`.

---

## 2. Docker / container config audit (if files exist)

Scan `docker-compose.yml`, `docker-compose.*.yml`, `Dockerfile`:

| Rule | Verdict |
|---|---|
| `environment:` section with `PASSWORD=` / `SECRET=` / `TOKEN=` and a bare string value | ❌ Must use `${ENV_VAR}` or Docker secrets |
| `ENV` in Dockerfile with a bare secret value | ❌ NEVER bake secrets into an image layer — use build args or runtime env |
| `ports:` mapping exposing sensitive services (DB:3306, Redis:6379) to `0.0.0.0` without a comment explaining why | ⚠️ WARNING — should be internal-only (`127.0.0.1:3306:3306`) unless intentional |

---

## 3. Vue / Vite config audit

### 3.1 `.env*` leak scan

- Scan all `.env*` **except** `.env.example`:
  - If you find `AWS_SECRET` / `STRIPE_SECRET_KEY` / `DB_*` / `JWT_*` / any secret **not** prefixed with `VITE_` → ❌ BLOCKER
  - If `.env` is tracked by git (`git ls-files .env`) → ❌ BLOCKER (should be in `.gitignore`)
- In `.env.development` / `.env.production`: check whether a production secret was accidentally placed in development
- **`.gitignore` completeness check**:
  - Verify `.env` and `.env.local` and `.env.*.local` appear in `.gitignore` → ❌ BLOCKER if missing
  - If `.gitignore` doesn't exist or is missing these entries, report it

### 3.2 `vite.config.ts` consistency

Check:
- `resolve.alias` has `@` → `src/` and `tsconfig.json` paths mirror it
- `build.sourcemap` is `false` for production (or at least flag it)
- Proxy rules that hardcode a localhost port that doesn't match `application.yml`'s `server.port`

> 📖 For detailed Vite env loading order, alias mirroring examples, and recommended scripts, read `references/vue-rules.md`.

### 3.3 `package.json` scripts conventions

- `dev` → `vite`
- `build` → `vite build`
- `preview` → `vite preview`
- Do `lint` / `type-check` scripts exist? If not, suggest:
  ```json
  "type-check": "vue-tsc --noEmit",
  "lint": "eslint . --ext .ts,.vue"
  ```

---

## 4. Summary report (required)

Output in this exact format:

```markdown
## 📋 Config Audit Report — {backend|frontend|all}

### ❌ Blockers (must fix — security risk / blocks deployment)
1. …

### ⚠️ Warnings (should fix — tech debt / unstable)
1. …

### ✅ Passed Checks
- …

### 📝 Action Items for Human
| # | Action | File(s) | Priority |
|---|--------|---------|----------|
| 1 | Replace DB password with ${DB_PASSWORD} | application-dev.yml | P0 |

(If everything is green → ✅ **ALL CLEAR — config compliance check passed**)
```

---

## 5. Repair mode (only when user explicitly says "fix it")

If the user explicitly says **"apply the suggestions"** / **"fix it"** / **"按建议修"** / **"帮我改"**:

1. **FIRST**: list every planned change as a numbered checklist — do NOT edit anything yet. Wait for user confirmation (this is required by CLAUDE.md).
2. After confirmation, make each `Edit` one at a time, outputting a diff summary after each.
3. **Never fill in a real password** — only write `${ENV_VAR:safe_default}` placeholders.
4. At the end, tell the user which env vars they need to add to CI / local `.env.local`.

---

### ⚡ Quick invocation

```
/check-config              ← full audit
/check-config backend      ← Spring only
/check-config frontend     ← Vue only
```
