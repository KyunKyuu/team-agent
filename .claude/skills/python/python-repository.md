---
name: python-repository
language: python
layer: repository
---

# Python Repository

## Key Patterns

- Accept `AsyncSession` (or `Session`) via constructor — never create a session inside the repo
- Define an abstract base class / Protocol so the service depends on the interface, not the impl
- Return model instances or `None`; raise nothing for "not found" — let the service decide
- Use `select()` + `scalars()` for typed queries (SQLAlchemy 2.x style)
- Paginate with `offset` / `limit`; return `(items, total)` tuple from list methods

## Example

```python
# app/repositories/user_repository.py
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from app.models.user import User

class UserRepository:
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def get_by_id(self, user_id: int) -> User | None:
        result = await self.session.execute(select(User).where(User.id == user_id))
        return result.scalar_one_or_none()

    async def get_by_email(self, email: str) -> User | None:
        result = await self.session.execute(select(User).where(User.email == email))
        return result.scalar_one_or_none()

    async def create(self, **kwargs) -> User:
        user = User(**kwargs)
        self.session.add(user)
        await self.session.flush()  # populate id without committing
        return user

    async def list(self, offset: int = 0, limit: int = 20) -> tuple[list[User], int]:
        items = (await self.session.execute(select(User).offset(offset).limit(limit))).scalars().all()
        total = (await self.session.execute(select(func.count(User.id)))).scalar_one()
        return list(items), total
```

## Checklist

- [ ] `AsyncSession` injected — repo never calls `sessionmaker` or `create_engine` itself
- [ ] No commit inside repository — caller (service or unit-of-work) commits the transaction
- [ ] `flush()` used after `add()` to get auto-generated IDs within the transaction
- [ ] All queries use `select()` ORM style, not raw SQL strings
- [ ] List methods accept and respect `offset`/`limit`; never return unbounded sets
- [ ] No business logic (no hashing, no validation) inside repository methods
- [ ] Repository tested with an in-memory or test DB using real sessions

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
