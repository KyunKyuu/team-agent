---
name: python-config
language: python
layer: config
---

# Python Config

## Key Patterns

- Use `pydantic-settings` (`BaseSettings`) for typed, validated config with env var loading
- Group settings into nested models (DatabaseSettings, AuthSettings, etc.)
- Instantiate a single settings object at module level; import it everywhere
- Fail fast: Pydantic raises `ValidationError` on startup if required vars are absent
- Use a `.env` file for local dev; real envs inject vars directly — no `.env` in production

## Example

```python
# app/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import PostgresDsn, field_validator

class DatabaseSettings(BaseSettings):
    url: PostgresDsn
    pool_size: int = 10

class AuthSettings(BaseSettings):
    jwt_secret: str
    jwt_expires_seconds: int = 3600

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_nested_delimiter="__")

    env: str = "development"
    port: int = 8000
    db: DatabaseSettings = DatabaseSettings()
    auth: AuthSettings = AuthSettings()

settings = Settings()
```

## Checklist

- [ ] `BaseSettings` used — not raw `os.getenv()` scattered through the codebase
- [ ] Required vars have no default and will raise `ValidationError` if unset
- [ ] Numeric/boolean fields typed correctly (`int`, `bool`), not `str`
- [ ] Settings object is a module-level singleton (`settings = Settings()`)
- [ ] `.env` listed in `.gitignore`; `.env.example` committed with placeholder values
- [ ] Nested settings use `env_nested_delimiter` or explicit `model_config`
- [ ] Startup test or smoke test imports `settings` to catch config errors early

---
> Customize this file for your specific stack (e.g., Sequelize vs TypeORM, Express vs Fastify).
