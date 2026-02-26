# База данных (Database)

См. оглавление: [Документация](README.md)

Компонент
- `khorsyio.db.database.Database` — обертка над SQLAlchemy AsyncEngine для простых операций и доступа к сессии
- DSN нормализуется к виду `postgresql+asyncpg://...` автоматически
- Параметры пула берутся из переменных окружения, см. [settings.md](settings.md)

Жизненный цикл
- `await app.db.connect()` — инициализация движка и фабрики сессий
- `await app.db.close()` — корректное закрытие движка
- Вызывается автоматически в `App.startup()`/`App.shutdown()`

Простые методы (совместимы со старым стилем)
- `fetch(sql: str, *args) -> list[dict]` — выборка множества строк
- `fetchrow(sql: str, *args) -> dict|None` — первая строка
- `fetchval(sql: str, *args)` — первое скалярное значение
- `execute(sql: str, *args) -> str` — модифицирующий запрос, возвращает строку вида `OK <rowcount>`
- `executemany(sql: str, args: list)` — пакетное выполнение

Плейсхолдеры
- Стиль `$1, $2, ...` автоматически переводится в именованные бинды `:p1, :p2, ...`
- Если в SQL уже есть именованные бинды, они сохраняются, позиционные аргументы мапятся на `:p1..`

Сессии SQLAlchemy
- Контекстный менеджер `async with app.db.session() as s: ...` возвращает `AsyncSession`
- На выходе выполняется `commit()`, при исключении выполняется `rollback()`

Пример
```python
from sqlalchemy import select

async with app.db.session() as s:
    result = await s.execute(select(User).where(User.active == True))
    users = result.scalars().all()
```

Дополнительно
- Утилиты фильтров/сортировки/пагинации: [query.md](query.md)
- Подробно об интеграции см. [INTEGRATION_SQLALCHEMY.md](INTEGRATION_SQLALCHEMY.md)
