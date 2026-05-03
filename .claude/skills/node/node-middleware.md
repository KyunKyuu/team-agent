---
name: node-middleware
language: node
layer: middleware
---

# Node Middleware

## Key Patterns

- Each middleware has a single responsibility — auth, logging, error handling are separate files
- JWT middleware: verify → attach decoded payload to `req.user` → call `next()`
- API key middleware: compare against config value with constant-time comparison
- Global error handler: last `app.use()`, signature `(err, req, res, next)` — maps domain errors to HTTP codes
- Never catch errors silently in middleware; always forward or respond

## Example

```ts
// src/middleware/auth.middleware.ts
import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";
import { config } from "../config";
import { UnauthorizedError } from "../errors";

export function authMiddleware(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.replace("Bearer ", "");
  if (!token) return next(new UnauthorizedError("Missing token"));

  try {
    req.user = jwt.verify(token, config.auth.jwtSecret) as JwtPayload;
    next();
  } catch {
    next(new UnauthorizedError("Invalid or expired token"));
  }
}

// src/middleware/error.middleware.ts
export function errorMiddleware(err: Error, req: Request, res: Response, next: NextFunction) {
  const status = (err as any).statusCode ?? 500;
  res.status(status).json({ success: false, message: err.message });
}
```

## Checklist

- [ ] JWT secret sourced from config, never hardcoded
- [ ] Token expiry validated by the JWT library (not manually)
- [ ] `req.user` typed via Express interface augmentation (`declare global`)
- [ ] API key comparison uses `crypto.timingSafeEqual` to prevent timing attacks
- [ ] Error middleware is the last `app.use()` call in app setup
- [ ] Error middleware maps known domain error types to correct HTTP status codes
- [ ] Middleware unit-tested by calling the function with mock `req`, `res`, `next`

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
