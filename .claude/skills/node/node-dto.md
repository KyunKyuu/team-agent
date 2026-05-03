---
name: node-dto
language: node
layer: dto
---

# Node DTO

## Key Patterns

- Validate at the boundary (handler/controller), not in service or model
- Use a schema library (Zod, Joi, class-validator) — one per project, not mixed
- Return structured errors: field name + message, not a single string
- Strip unknown fields to prevent mass-assignment vulnerabilities
- Keep request DTO and response DTO separate; response shapes should be explicit

## Example

```ts
// src/dto/create-user.dto.ts  (Zod style)
import { z } from "zod";

export const CreateUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(128),
  fullName: z.string().min(1).max(255),
});

export type CreateUserDto = z.infer<typeof CreateUserSchema>;

// Validation helper
export function validateCreateUser(body: unknown): CreateUserDto {
  return CreateUserSchema.parse(body); // throws ZodError on failure
}

// Response DTO (explicit shape, no password leak)
export interface UserResponse {
  id: string;
  email: string;
  fullName: string;
  createdAt: string;
}
```

## Checklist

- [ ] Schema defined separately from handler — importable and testable alone
- [ ] All required fields marked required; optional fields have defaults or `optional()`
- [ ] String fields have `min`/`max` length constraints
- [ ] Unknown/extra fields are stripped or rejected
- [ ] Response DTO never exposes sensitive fields (password hashes, internal IDs)
- [ ] Validation errors return HTTP 400 with field-level detail
- [ ] DTOs covered by unit tests with valid and invalid fixture inputs

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
