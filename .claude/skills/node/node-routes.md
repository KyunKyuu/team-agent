---
name: node-routes
language: node
layer: routes
---

# Node Routes

## Key Patterns

- Group routes by resource in separate router files; mount under a versioned prefix
- Route files only wire paths to handlers — no logic, no middleware inline
- Attach resource-level middleware (auth, rate-limit) at the router level, not per-route
- Use HTTP verbs semantically: GET (read), POST (create), PUT/PATCH (update), DELETE (remove)
- Export a factory function that receives the handler instance to enable DI

## Example

```ts
// src/routes/user.routes.ts  (Express style)
import { Router } from "express";
import { UserHandler } from "../handlers/user.handler";
import { authMiddleware } from "../middleware/auth.middleware";

export function userRoutes(handler: UserHandler): Router {
  const router = Router();

  router.post("/", handler.create);
  router.get("/:id", authMiddleware, handler.getById);
  router.patch("/:id", authMiddleware, handler.update);
  router.delete("/:id", authMiddleware, handler.remove);

  return router;
}

// src/routes/index.ts
import { Router } from "express";
import { userRoutes } from "./user.routes";

export function registerRoutes(deps: AppDependencies): Router {
  const root = Router();
  root.use("/users", userRoutes(deps.userHandler));
  return root;
}
```

## Checklist

- [ ] All routes grouped under `/api/v{n}/` or equivalent versioned prefix
- [ ] Route files are pure wiring — zero logic, zero DB calls
- [ ] Auth middleware applied at router level for protected resource groups
- [ ] Path parameters use consistent naming (`id`, not `userId` in some, `id` in others)
- [ ] Route factory accepts handler via parameter (not importing a singleton)
- [ ] 404 handler registered last in the app after all route mounts
- [ ] Route list is documented or auto-generated (e.g. Swagger/OpenAPI)

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
