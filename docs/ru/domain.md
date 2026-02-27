# Домены (Domain) и неймспейсы

См. оглавление: [Документация](README.md)

Назначение
- Объединение обработчиков событий и HTTP-маршрутов в функциональные модули
- Автоматическая префиксация типов событий через `namespace`
- Упрощенный DI при создании обработчиков

Ключевые поля
- `handlers: list` — классы или экземпляры `Handler`
- `routes: list[Route]` — HTTP-маршруты
- `namespace: str` — префикс для `subscribes_to` и `publishes`

Поведение `setup(app)`
- Для каждого элемента из `handlers`
  - если это класс, он инстанцируется с DI по именам параметров конструктора
  - если это экземпляр, используется как есть
  - если задан `namespace`, к `subscribes_to` и `publishes` добавится префикс `"{namespace}."`
  - хендлер регистрируется в `app.bus`
- Маршруты монтируются в `app.router`

Пример
```python
from khorsyio import Domain, Route

users = Domain()
users.namespace = "user"
users.handlers = [CreateUser]
users.routes = [Route("GET", "/users", get_users)]

app.mount(users)
```

DI соответствие имен
- `db -> app.db`
- `client -> app.client`
- `bus -> app.bus`
- `transport -> app.transport`
- `app -> app`
- другое имя -> `app`

Связанные разделы
- [Handlers](handlers.md)
- [HTTP](http.md)
- [App](app.md)
