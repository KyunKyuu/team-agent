---
name: python-middleware
language: python
layer: middleware
---

# Python Middleware

## Key Patterns

- JWT auth as a FastAPI dependency (`Depends`), not ASGI middleware — easier to apply per-route
- API key auth follows the same pattern: extract header → validate → raise `HTTPException(401)`
- CORS configured via `CORSMiddleware` added to the app — never handled manually in routes
- Global exception handler registered with `@app.exception_handler` — maps domain errors to HTTP
- Avoid stateful middleware; pass context via `request.state` if needed within a request

## Example

```python
# app/dependencies/auth.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
from app.config import settings

bearer = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(bearer),
):
    try:
        payload = jwt.decode(credentials.credentials, settings.auth.jwt_secret, algorithms=["HS256"])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")

# app/main.py  — CORS + exception handler wiring
from fastapi.middleware.cors import CORSMiddleware
from app.core.exceptions import NotFoundError, ConflictError

app.add_middleware(CORSMiddleware, allow_origins=settings.cors_origins,
                   allow_methods=["*"], allow_headers=["*"])

@app.exception_handler(NotFoundError)
async def not_found_handler(request, exc):
    return JSONResponse(status_code=404, content={"detail": str(exc)})
```

## Checklist

- [ ] JWT secret sourced from `settings`, never hardcoded
- [ ] Token expiry validated by the JWT library — not manually checked
- [ ] `HTTPBearer` (or `OAuth2PasswordBearer`) used as the FastAPI security scheme
- [ ] CORS origins configured from settings, not hardcoded `"*"` in production
- [ ] Exception handlers registered for each domain error type
- [ ] API key comparison uses `secrets.compare_digest` to prevent timing attacks
- [ ] Auth dependency applied at router level for protected resource groups

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
