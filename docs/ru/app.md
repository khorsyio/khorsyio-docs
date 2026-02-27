# Приложение (App)

См. оглавление: [Документация](README.md)

App объединяет
- Шину событий: `app.bus`
- HTTP-приложение и роутер: `app.router`
- WebSocket транспорт: `app.transport`
- Базу данных: `app.db`
- Внешний HTTP-клиент: `app.client`

Жизненный цикл
- `startup()` подключает БД, запускает HttpClient, валидирует граф хендлеров, стартует Bus
- `shutdown()` останавливает Bus, закрывает HttpClient и БД
- `run()` запускает ASGI сервер (granian, иначе uvicorn)

Регистрация
- `register(handler)` регистрирует экземпляр хендлера в шине
- `mount(domain)` монтирует Domain: добавляет хендлеры/роуты и префикс namespace

ASGI
- `__call__` обрабатывает `lifespan`, `http` и `websocket`

Связанные разделы
- [Handlers](handlers.md)
- [Domain](domain.md)
- [HTTP](http.md)
- [Bus](bus.md)
- [Транспорт](transport.md)
- [Database](db.md)
- [HttpClient](client.md)
- [Настройки](settings.md)
