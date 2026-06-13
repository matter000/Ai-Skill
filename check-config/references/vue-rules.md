# Vue / Vite Configuration Rules — Detailed Reference

> Loaded when the skill needs deeper Vue/Vite rules or the user asks for detail.

## `.env*` file discipline

### What Vite exposes

Only variables prefixed with `VITE_` are injected into client code at build time. Everything else is ignored by Vite — but **still visible in the codebase** if the file is committed.

### Vite env loading order

1. `.env` — shared defaults, loaded in all modes
2. `.env.local` — local overrides, **gitignored**
3. `.env.[mode]` — mode-specific (e.g., `.env.development`)
4. `.env.[mode].local` — mode-specific local overrides, **gitignored**

### Correct setup

```ini
# .env — committed, public defaults only
# Shared defaults for all modes
VITE_APP_TITLE=My Application
VITE_API_TIMEOUT=10000

# .env.development — committed, dev overrides
# Development environment overrides
VITE_API_BASE_URL=http://localhost:8080/api

# .env.production — committed, prod overrides
# Production environment settings
VITE_API_BASE_URL=https://api.example.com

# .env.local — NEVER committed, secrets
VITE_SENTRY_DSN=https://xxxxx@xxxxx.ingest.sentry.io/xxxxx
```

### Red flags

| Pattern | Risk | Action |
|---------|------|--------|
| `SECRET_KEY=` (no `VITE_` prefix) | Won't work in Vite; but if accidentally prefixed later, leaks | Remove or prefix correctly; keep in `.env.local` |
| `VITE_STRIPE_SECRET=` | Will be baked into the JS bundle — anyone can see it in DevTools | Move to backend; frontend only stores `VITE_STRIPE_PUBLISHABLE_KEY` |
| `.env` is git-tracked (`git ls-files .env` returns it) | `.env` should be in `.gitignore` | Add `.env` to `.gitignore`; only commit `.env.example` |

## `vite.config.ts` conventions

### Path alias consistency

Both files must agree:

```ts
// vite.config.ts
resolve: {
  alias: {
    '@': fileURLToPath(new URL('./src', import.meta.url)),
  },
}
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

### Proxy configuration

Each proxy rule must have a comment explaining which backend service it targets:

```ts
server: {
  proxy: {
    // Gateway: all /api requests go to Spring Boot backend
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true,
    },
  },
}
```

### Build flags

```ts
build: {
  sourcemap: false,    // must be false for production
  minify: 'terser',    // or 'esbuild'
}
```

## `tsconfig.json` checks

| Rule | Severity |
|------|----------|
| `strict: true` must not be downgraded to `false` | ❌ BLOCKER |
| `compilerOptions.paths` must mirror `vite.config.ts` `resolve.alias` | ⚠️ WARNING |
| `noUnusedLocals: true` recommended but not required | ℹ️ INFO |

## `package.json` scripts

### Recommended scripts checklist

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "type-check": "vue-tsc --noEmit",
    "lint": "eslint . --ext .ts,.vue",
    "lint:fix": "eslint . --ext .ts,.vue --fix",
    "format": "prettier --write \"src/**/*.{ts,vue,css}\""
  }
}
```

- `type-check` catches template type errors without emitting files
- `lint` ensures code style consistency
- Missing both → suggest adding at minimum `type-check`
