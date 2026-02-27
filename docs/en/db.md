# Database

See: [Documentation](README.md)

Component
- `khorsyio.db.database.Database` — wrapper over SQLAlchemy AsyncEngine for simple operations and session access
- DSN is normalized to `postgresql+asyncpg://...` automatically
- Pool parameters are taken from environment variables, see [settings.md](settings.md)

Lifecycle
- `await app.db.connect()` — initializes the engine and session factory
- `await app.db.close()` — cleanly closes the engine
- Called automatically in `App.startup()` and `App.shutdown()`

Simple methods (compatible with legacy asyncpg style)
- `fetch(sql: str, *args) -> list[dict]` — fetch multiple rows
- `fetchrow(sql: str, *args) -> dict|None` — first row
- `fetchval(sql: str, *args)` — first scalar value
- `execute(sql: str, *args) -> str` — modifying query, returns a string like `OK <rowcount>`
- `executemany(sql: str, args: list)` — batch execution

Placeholders
- `$1, $2, ...` style is automatically converted to named binds `:p1, :p2, ...`
- If the SQL already has named binds they are preserved; positional arguments are mapped to `:p1...`

SQLAlchemy sessions
- Context manager `async with app.db.session() as s: ...` returns `AsyncSession`
- On exit `commit()` is called; on exception `rollback()` is called

Example
```python
from sqlalchemy import select

async with app.db.session() as s:
    result = await s.execute(select(User).where(User.active == True))
    users = result.scalars().all()
```

Related sections
- [Query utilities](query.md)
- [Settings](settings.md)
