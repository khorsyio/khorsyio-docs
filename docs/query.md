# Запросы SQLAlchemy: фильтры, сортировка, пагинация, курсор

См. оглавление: [Документация](README.md)

Утилиты находятся в `khorsyio.db.query`
- `apply_filters(stmt, Model, filters: dict | None) -> Select`
- `apply_order(stmt, Model, order: Sequence[str] | str | None) -> Select`
- `paginate(session, stmt, page: int = 1, per_page: int = 20) -> dict`
- `apply_cursor(stmt, Model, cursor_value: Any | None, cursor_field: str = "id", order: str = "asc") -> Select`

Фильтры `apply_filters`
- Без суфикса — точное сравнение: `{ "name": "Bob" }`
- С операторами: ключ `field__op`
  - Поддерживаются: `eq, ne, lt, lte, gt, gte, in, contains, icontains, startswith, istartswith, endswith, iendswith, isnull, between`
  - Пример: `{ "age__gte": 18, "name__icontains": "alex" }`

Сортировка `apply_order`
- Принимает строку или список строк
- Префикс `-` для убывающей сортировки, `+` или без префикса — по возрастанию
- Пример: `["-created_at", "id"]`

Пагинация `paginate`
- Выполняет подсчет total, возвращает структуру `{ items, total, page, per_page, pages }`
- Если запрос выбирает одну сущность/колонку — `items` будет списком ORM-экземпляров, иначе — списком dict

Курсор `apply_cursor`
- Добавляет ограничение по монотонно растущему полю, например `id` или `created_at`
- Пример использования для бесконечной прокрутки

Пример сквозного использования
```python
from sqlalchemy import select
from khorsyio.db.query import apply_filters, apply_order, paginate, apply_cursor

stmt = select(User)
stmt = apply_filters(stmt, User, {"active": True, "age__gte": 18})
stmt = apply_order(stmt, User, ["-created_at"])  # DESC по created_at
stmt = apply_cursor(stmt, User, cursor_value=last_id, cursor_field="id", order="asc")

async with app.db.session() as s:
    page = await paginate(s, stmt, page=1, per_page=20)
```

Связанные разделы
- [База данных](db.md)
- [Интеграция SQLAlchemy](INTEGRATION_SQLALCHEMY.md)
