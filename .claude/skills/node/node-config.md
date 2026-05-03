---
name: node-config
language: node
layer: config
---

# Node Config

## Key Patterns

- Load env vars via `dotenv` at process entry point, never inside modules
- Export a single validated config object — callers import the object, not `process.env`
- Fail fast: throw on startup if required vars are missing
- Separate config per concern (db, server, auth, external APIs)
- Use typed accessors so misconfigured vars surface at compile/lint time (TypeScript)

## Example

```ts
// src/config/index.ts
import "dotenv/config";

function require(key: string): string {
  const val = process.env[key];
  if (!val) throw new Error(`Missing required env var: ${key}`);
  return val;
}

export const config = {
  server: {
    port: Number(process.env.PORT ?? 3000),
    env: process.env.NODE_ENV ?? "development",
  },
  db: {
    url: require("DATABASE_URL"),
    pool: Number(process.env.DB_POOL ?? 10),
  },
  auth: {
    jwtSecret: require("JWT_SECRET"),
    jwtExpiresIn: process.env.JWT_EXPIRES_IN ?? "1h",
  },
};
```

## Checklist

- [ ] `dotenv/config` imported once at entry point (e.g. `src/index.ts`)
- [ ] All required vars guarded with a throw on missing value
- [ ] No raw `process.env.*` access outside `config/` directory
- [ ] Numeric/boolean vars explicitly cast from string
- [ ] `.env.example` committed; `.env` in `.gitignore`
- [ ] Config object exported as a single named const, not default
- [ ] Unit test or startup smoke-test validates config loading

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
