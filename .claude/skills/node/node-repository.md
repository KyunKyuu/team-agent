---
name: node-repository
language: node
layer: repository
---

# Node Repository

## Key Patterns

- One repository class per aggregate root — no cross-entity queries inside a repo
- Accept the ORM client/data-source via constructor injection (enables testing with fakes)
- Return domain objects or plain objects, never raw ORM instances to the service layer
- Encapsulate all query builder logic here — services call named methods, not raw queries
- Paginate via offset/limit or cursor; never return unbounded result sets

## Example

```ts
// src/repositories/user.repository.ts
import { DataSource, Repository } from "typeorm";
import { User } from "../models/user.model";

export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: Partial<User>): Promise<User>;
  delete(id: string): Promise<void>;
}

export class UserRepository implements IUserRepository {
  private repo: Repository<User>;

  constructor(dataSource: DataSource) {
    this.repo = dataSource.getRepository(User);
  }

  findById(id: string) {
    return this.repo.findOneBy({ id });
  }

  findByEmail(email: string) {
    return this.repo.findOneBy({ email });
  }

  save(data: Partial<User>) {
    return this.repo.save(data);
  }

  async delete(id: string) {
    await this.repo.delete(id);
  }
}
```

## Checklist

- [ ] Interface defined alongside class — service depends on interface, not concrete class
- [ ] DataSource/client injected via constructor, not imported directly
- [ ] No business logic (no validation, no hashing) inside repository methods
- [ ] All finder methods return `null` (not throw) when record not found
- [ ] List methods accept pagination params and return `{ data, total }` or equivalent
- [ ] Transactions handled via passed-in manager/session, not started inside repo
- [ ] Repository covered by integration tests against a real (test) DB

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
