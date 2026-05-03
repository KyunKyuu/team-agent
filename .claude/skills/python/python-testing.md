---
name: python-testing
language: python
layer: testing
---

# Python Testing

## Key Patterns

- Use `pytest` with `pytest-asyncio` for async tests; mark async tests with `@pytest.mark.asyncio`
- HTTP integration tests via FastAPI `TestClient` (sync) or `AsyncClient` from `httpx`
- Fixtures in `conftest.py` — one shared DB session fixture rolled back per test
- Mock external I/O (email, S3, third-party APIs) with `unittest.mock.patch` or `pytest-mock`
- Arrange-Act-Assert in every test; one logical assertion per test

## Example

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from app.main import app
from app.db.base import Base

@pytest.fixture(scope="session")
async def engine():
    eng = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with eng.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield eng
    await eng.dispose()

@pytest.fixture
async def session(engine):
    async with engine.begin() as conn:
        async with AsyncSession(bind=conn) as s:
            yield s
            await conn.rollback()  # clean up after each test

@pytest.fixture
async def client():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c

# tests/test_user_router.py
@pytest.mark.asyncio
async def test_create_user_returns_201(client):
    response = await client.post("/api/v1/users/", json={"email": "a@b.com", "password": "pass1234", "full_name": "A"})
    assert response.status_code == 201
    assert response.json()["email"] == "a@b.com"
```

## Checklist

- [ ] `conftest.py` at project root with shared `engine`, `session`, and `client` fixtures
- [ ] DB session fixture rolls back or drops data after each test — no state bleed
- [ ] Async tests decorated with `@pytest.mark.asyncio` (or `asyncio_mode = "auto"` in config)
- [ ] Service unit tests mock the repository — no real DB needed
- [ ] HTTP tests use `AsyncClient` with `ASGITransport` — no running server needed
- [ ] `.env.test` used for test config (separate test DB URL)
- [ ] `pytest --cov` runs in CI with a minimum coverage threshold

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
