---
name: python-model
language: python
layer: model
---

# Python Model

## Key Patterns

- Inherit from a shared `Base = DeclarativeBase()` defined once in `app/db/base.py`
- Use `Mapped[T]` and `mapped_column()` (SQLAlchemy 2.x) for fully typed columns
- Declare timestamps with `server_default=func.now()` so DB owns the value
- Define `__tablename__` explicitly — never rely on auto-naming
- Keep relationships on both sides; use `lazy="selectin"` for async-safe loading

## Example

```python
# app/models/user.py
from datetime import datetime
from sqlalchemy import String, Boolean, func
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.db.base import Base

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    password_hash: Mapped[str] = mapped_column(String(255), nullable=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now(), nullable=False)
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), onupdate=func.now(), nullable=False
    )
```

## Checklist

- [ ] `__tablename__` explicitly defined
- [ ] All columns use `Mapped[T]` with appropriate Python type
- [ ] Primary key strategy consistent with schema (int, uuid, etc.)
- [ ] Timestamps use `server_default` (DB-side), not Python `datetime.utcnow`
- [ ] Sensitive fields (passwords) are hashes only — no plain-text columns
- [ ] Model imported into `app/db/base.py` so Alembic detects it for migrations
- [ ] No business methods on the model — pure data structure only

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
