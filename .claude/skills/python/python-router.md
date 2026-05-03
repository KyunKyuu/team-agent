---
name: python-router
language: python
layer: routes
---

# Python Router

## Key Patterns

- Use `APIRouter` with a `prefix` and `tags` per resource; include into the main `app` once
- Inject services via `Depends()` — keep router functions thin (parse → call service → return schema)
- Declare response model on every route using Pydantic response schemas
- Use `status_code` explicitly: 201 for create, 204 for delete, 200 for everything else
- Route functions must not contain business logic — one `await service.method()` call per route

## Example

```python
# app/routers/user.py
from fastapi import APIRouter, Depends, status
from app.schemas.user import CreateUserRequest, UserResponse
from app.services.user_service import UserService
from app.dependencies import get_user_service

router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    body: CreateUserRequest,
    service: UserService = Depends(get_user_service),
):
    return await service.create(body)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service),
):
    return await service.get_by_id(user_id)
```

## Checklist

- [ ] `APIRouter` has `prefix` and `tags` defined
- [ ] Every route declares `response_model` with a Pydantic schema
- [ ] Service injected via `Depends()`, not instantiated inside the route function
- [ ] `status_code` explicitly set for non-200 responses
- [ ] Route function body is 1-3 lines: validate path/query params → call service → return
- [ ] Router included in `app/main.py` under a versioned prefix (`/api/v1`)
- [ ] Domain exceptions from service layer caught by a registered exception handler

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
