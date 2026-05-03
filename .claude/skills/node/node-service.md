---
name: node-service
language: node
layer: service
---

# Node Service

## Key Patterns

- Services own all business logic; handlers and repositories own none
- Receive dependencies (repositories, other services, config) via constructor — no `new` inside
- Throw typed domain errors (e.g. `NotFoundError`, `ConflictError`) — let middleware map to HTTP
- Never import Express/Fastify types — service layer is transport-agnostic
- Keep methods small: one public method = one business operation

## Example

```ts
// src/services/user.service.ts
import { IUserRepository } from "../repositories/user.repository";
import { CreateUserDto } from "../dto/create-user.dto";
import { ConflictError, NotFoundError } from "../errors";
import bcrypt from "bcrypt";

export class UserService {
  constructor(private userRepo: IUserRepository) {}

  async create(dto: CreateUserDto) {
    const existing = await this.userRepo.findByEmail(dto.email);
    if (existing) throw new ConflictError("Email already in use");

    const passwordHash = await bcrypt.hash(dto.password, 12);
    return this.userRepo.save({ ...dto, passwordHash });
  }

  async getById(id: string) {
    const user = await this.userRepo.findById(id);
    if (!user) throw new NotFoundError("User not found");
    return user;
  }
}
```

## Checklist

- [ ] No HTTP request/response objects (`req`, `res`) imported or used
- [ ] All dependencies injected — class is instantiable in unit tests with mocks
- [ ] Domain errors thrown with clear messages; not raw `Error` or HTTP status codes
- [ ] Side effects (email, events, cache invalidation) happen here, not in the repository
- [ ] Async methods all `await`-ed; no unhandled promise chains
- [ ] Unit tests mock the repository and assert service behavior in isolation
- [ ] Methods return plain data objects, not ORM entity instances

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
