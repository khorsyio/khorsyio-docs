# HTTP: Router, Request, Response, CORS

См. оглавление: [Документация](README.md)

Компоненты
- Request — обертка над ASGI-сообщением для чтения тела, заголовков, куки и параметров
- Response — утилиты для отправки ответа (JSON/текст/ошибка)
- Router — регистрация и разрешение маршрутов с поддержкой path-параметров
- CorsConfig — конфигурация CORS
- HttpApp — ASGI-приложение, применяет middleware-хуки и CORS

Request
- `cookies()` -> dict
- `cookie(key, default=None)`
- `header(key, default=None)`
- `body()` -> bytes
- `json(typ: type|None=None)` -> декод JSON, при указании `typ` используется msgspec для валидации
- `param(key, default=None)` — query string параметр

Response
- `json(send, data, status=200, headers=None, cookies=None)` — отправляет application/json
- `ok(send, **kwargs)` — шорткат для JSON-ответа 200 с произвольными ключами
- `text(send, text, status=200, headers=None, cookies=None)`
- `error(send, message, status=400, code="error")` — JSON с полями `code` и `message`

Router
- `add(method, path, handler)` — регистрация обработчика
- `get/post/put/delete(path, handler)` — шорткаты
- `use(middleware)` — до-обработчик, сигнатура `(req, send, next)` должен вызвать `await next()`
- `after(hook)` — пост-хук, сигнатура `(req, res)` где `res` — словарь отправленного ответа
- `resolve(method, path)` — используется внутренне HttpApp
- Поддержка path-параметров вида `/users/{id}` со сбором в `req.scope["path_params"]`

CorsConfig
- `origins` список или `"*"`
- `methods`, `headers`, `credentials`, `max_age`
- `allowed_origin(origin)` — проверяет домен

HttpApp
- Конструируется как `HttpApp(router, cors=None)`
- Является ASGI-приложением: `await http(scope, receive, send)`
- Обрабатывает `OPTIONS` preflight при включенном CORS
- Добавляет заголовки CORS при ответе
- Ведет минимальный трекинг запроса

Пример
```python
from khorsyio import App, Response

app = App()

async def health(req, send):
    await Response.ok(send, status="ok")

app.router.get("/health", health)
```

Связанные разделы
- [App](app.md)
- [События](events.md)
- [Handlers](handlers.md)
- [Домены](domain.md)
