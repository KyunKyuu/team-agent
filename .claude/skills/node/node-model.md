---
name: node-model
language: node
layer: model
---

# Node Model

## Key Patterns

- Define model/entity in one file; keep DB concerns out of business logic
- Use decorators (TypeORM) or `define`/`init` (Sequelize) — pick one ORM and be consistent
- Declare timestamps (`createdAt`, `updatedAt`) explicitly even if ORM auto-adds them
- Map column names to DB snake_case; expose camelCase to application layer
- Keep associations/relations declared at the model level, not scattered in queries

## Example

```ts
// src/models/user.model.ts  (TypeORM style)
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, UpdateDateColumn } from "typeorm";

@Entity("users")
export class User {
  @PrimaryGeneratedColumn("uuid")
  id: string;

  @Column({ unique: true, length: 255 })
  email: string;

  @Column({ name: "password_hash", length: 255 })
  passwordHash: string;

  @Column({ name: "is_active", default: true })
  isActive: boolean;

  @CreateDateColumn({ name: "created_at" })
  createdAt: Date;

  @UpdateDateColumn({ name: "updated_at" })
  updatedAt: Date;
}
```

## Checklist

- [ ] Table/collection name explicitly declared (not inferred from class name)
- [ ] Primary key type consistent with DB schema (uuid / int / bigint)
- [ ] Sensitive columns (passwords) stored as hash, never plain text
- [ ] Timestamps present and use DB-level defaults where possible
- [ ] No business logic or HTTP references inside the model file
- [ ] Associations defined with correct cascade and nullable settings
- [ ] Model exported and registered in ORM config/data-source

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
