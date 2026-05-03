---
name: python-service
language: python
layer: service
---

# Python Service

## Key Patterns

- Services own all business logic; routers and repositories own none
- Accept repository (and other services) via constructor for dependency injection
- Raise domain-specific exceptions (`UserNotFoundError`, `DuplicateEmailError`) — router maps to HTTP
- Commit the transaction at the service boundary, not inside the repository
- Keep service methods async to align with FastAPI's async request handling

## Example

```python
# app/services/user_service.py
from app.repositories.user_repository import UserRepository
from app.schemas.user import CreateUserRequest
from app.core.security import hash_password
from app.core.exceptions import NotFoundError, ConflictError
from sqlalchemy.ext.asyncio import AsyncSession

class UserService:
    def __init__(self, session: AsyncSession) -> None:
        self.repo = UserRepository(session)
        self.session = session

    async def create(self, dto: CreateUserRequest):
        if await self.repo.get_by_email(dto.email):
            raise ConflictError("Email already in use")
        user = await self.repo.create(
            email=dto.email,
            password_hash=hash_password(dto.password),
            full_name=dto.full_name,
        )
        await self.session.commit()
        await self.session.refresh(user)
        return user

    async def get_by_id(self, user_id: int):
        user = await self.repo.get_by_id(user_id)
        if not user:
            raise NotFoundError("User not found")
        return user
```

## Checklist

- [ ] No FastAPI `Request`/`Response` objects imported or used
- [ ] Dependencies (repo, session) injected — no module-level singletons
- [ ] Domain exceptions raised with descriptive messages; not raw `Exception`
- [ ] `session.commit()` called after all writes in a single operation
- [ ] `session.refresh(obj)` called after commit to reload server-set fields
- [ ] Side effects (emails, events) triggered here, after successful commit
- [ ] Unit tests mock the repository and assert service logic in isolation

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
