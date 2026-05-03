---
name: node-handler
language: node
layer: handler
---

# Node Handler

## Key Patterns

- Handlers are thin: parse → validate → call service → serialize response
- No business logic in handlers — delegate immediately to the service layer
- Use a central error handler middleware; handlers should not catch-and-swallow errors
- Consistent response envelope: `{ success, data, message }` or align with API contract
- Extract and validate path/query params the same way as body — same DTO/schema approach

## Example

```ts
// src/handlers/user.handler.ts  (Express style)
import { Request, Response, NextFunction } from "express";
import { UserService } from "../services/user.service";
import { validateCreateUser } from "../dto/create-user.dto";

export class UserHandler {
  constructor(private userService: UserService) {}

  create = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const dto = validateCreateUser(req.body);
      const user = await this.userService.create(dto);
      res.status(201).json({ success: true, data: user });
    } catch (err) {
      next(err); // delegate to error middleware
    }
  };

  getById = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const user = await this.userService.getById(req.params.id);
      res.json({ success: true, data: user });
    } catch (err) {
      next(err);
    }
  };
}
```

## Checklist

- [ ] Handler methods contain no business logic — one service call per handler
- [ ] Input validated before service call; invalid input returns 400 immediately
- [ ] Errors forwarded via `next(err)` (Express) or equivalent, not swallowed
- [ ] HTTP status codes are meaningful (201 for create, 204 for delete, etc.)
- [ ] Response shape is consistent with API contract / other handlers
- [ ] Handler tested via supertest / injection — not unit tested with mocked res/req
- [ ] No direct DB or repository calls inside the handler

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
