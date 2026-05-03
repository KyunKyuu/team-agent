---
name: python-schema
language: python
layer: dto
---

# Python Schema

## Key Patterns

- Use Pydantic v2 `BaseModel` for all request and response shapes
- Keep request schema and response schema as separate classes — never reuse the same model for both
- Use `model_config = ConfigDict(from_attributes=True)` on response schemas to enable ORM mapping
- Add field validators (`@field_validator`) for custom rules; prefer built-in types (`EmailStr`, `HttpUrl`)
- Response schemas must never expose sensitive fields (password hashes, internal tokens)

## Example

```python
# app/schemas/user.py
from pydantic import BaseModel, EmailStr, ConfigDict, field_validator

class CreateUserRequest(BaseModel):
    email: EmailStr
    password: str
    full_name: str

    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError("Password must be at least 8 characters")
        return v

class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: str
    full_name: str
    is_active: bool
```

## Checklist

- [ ] Request schema validates all user input with explicit field constraints
- [ ] Response schema has `from_attributes=True` when built from ORM objects
- [ ] Sensitive fields absent from response schema (no `password_hash`)
- [ ] Optional fields use `field: T | None = None` syntax (Pydantic v2)
- [ ] Custom validators use `@field_validator` (not deprecated `@validator`)
- [ ] Schemas importable and testable independently from FastAPI
- [ ] Schemas used as FastAPI type hints — automatic OpenAPI docs generated

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
