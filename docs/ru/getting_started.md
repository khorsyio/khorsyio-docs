# Быстрый старт

См. также оглавление: [Документация](README.md)

Минимальное приложение

```python
from khorsyio import App, Response, CorsConfig

app = App(cors=CorsConfig())
app.router.get("/health", lambda req, send: Response.ok(send, status="ok"))

if __name__ == "__main__":
    app.run()
```

- Как это работает
  - App объединяет HTTP, шину событий и транспорт WebSocket
  - Router разрешает маршрут и вызывает ваш обработчик
  - Response.ok отправляет JSON с кодом 200

Дальше по темам
- Архитектура: [architecture.md](architecture.md)
- HTTP API: [http.md](http.md)
- Шина событий: [bus.md](bus.md)
- События и структуры: [events.md](events.md)
- Обработчики: [handlers.md](handlers.md)
- Домены и DI: [domain.md](domain.md)
- Транспорт WebSocket: [transport.md](transport.md)
- Клиент HTTP: [client.md](client.md)
- База данных: [db.md](db.md)
- Утилиты запросов SQLAlchemy: [query.md](query.md)
- Настройки окружения: [settings.md](settings.md)
