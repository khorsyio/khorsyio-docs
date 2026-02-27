# SQLAlchemy Queries: filters, sorting, pagination, cursor

See: [Documentation](README.md)

Utilities are in `khorsyio.db.query`
- `apply_filters(stmt, Model, filters: dict | None) -> Select`
- `apply_order(stmt, Model, order: Sequence[str] | str | None) -> Select`
- `paginate(session, stmt, page: int = 1, per_page: int = 20) -> dict`
- `apply_cursor(stmt, Model, cursor_value: Any | None, cursor_field: str = "id", order: str = "asc") -> Select`

Filters `apply_filters`
- No suffix — exact comparison: `{ "name": "Bob" }`
- With operators: key `field__op`
  - Supported: `eq, ne, lt, lte, gt, gte, in, contains, icontains, startswith, istartswith, endswith, iendswith, isnull, between`
  - Example: `{ "age__gte": 18, "name__icontains": "alex" }`

Sorting `apply_order`
- Accepts a string or a list of strings
- Prefix `-` for descending, `+` or no prefix for ascending
- Example: `["-created_at", "id"]`

Pagination `paginate`
- Runs a total count, returns `{ items, total, page, per_page, pages }`
- If the query selects a single entity or column, `items` is a list of ORM instances; otherwise a list of dicts

Cursor `apply_cursor`
- Adds a constraint on a monotonically increasing field such as `id` or `created_at`
- Useful for infinite scroll

Full usage example
```python
from sqlalchemy import select
from khorsyio.db.query import apply_filters, apply_order, paginate, apply_cursor

stmt = select(User)
stmt = apply_filters(stmt, User, {"active": True, "age__gte": 18})
stmt = apply_order(stmt, User, ["-created_at"])
stmt = apply_cursor(stmt, User, cursor_value=last_id, cursor_field="id", order="asc")

async with app.db.session() as s:
    page = await paginate(s, stmt, page=1, per_page=20)
```

Related sections
- [Database](db.md)
