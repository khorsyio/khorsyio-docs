# Обзор архитектуры

См. оглавление: [Документация](README.md)

Основные элементы
- App — точка входа, собирает все подсистемы, управляет жизненным циклом
- HTTP — Router/Request/Response/CorsConfig, простой ASGI-приложение с middleware-хуками
- Bus — внутренняя событийная шина с очередями, таймаутами и обработкой ошибок
- Structs — Context, Envelope, Error, конвенция сериализации через msgspec
- Handlers — базовый класс и контракт для обработки событий
- Domain — группировка хендлеров и HTTP-маршрутов + namespace/DI
- Transport — SocketTransport (ASGI) и SocketClient (клиент socket.io)
- Client — HttpClient на httpx для внешних HTTP-запросов
- DB — Database (SQLAlchemy async engine) и query-хелперы

Поток запроса HTTP
1) ASGI попадает в HttpApp
2) Router находит хендлер по методу/пути
3) Ваш хендлер работает с Request и отправляет Response

Поток события
1) Источник публикует Envelope в Bus (publish или request)
2) Bus диспетчит на подходящий Handler
3) Handler возвращает новый Envelope или None
4) Bus продолжает обработку или отвечает инициатору

Связанные разделы
- [App](app.md)
- [HTTP](http.md)
- [Bus](bus.md) и [События](events.md)
- [Handlers](handlers.md), [Domain](domain.md)
- [Transport](transport.md)
- [Database](db.md), [Query-хелперы](query.md)

## Известные ограничения (Troubleshooting)
- **JWT**: Поле `sub` обязано быть строкой для `PyJWT >= 2.0.0` (не `int`).
