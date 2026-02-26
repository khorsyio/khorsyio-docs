# Транспорт WebSocket (SocketTransport/Client)

См. оглавление: [Документация](README.md)

SocketTransport
- Назначение: интеграция socket.io (asgi) для двунаправленных событий
- Конструктор: `SocketTransport(bus: Bus, async_mode: str = "asgi")`
- Внутри поднимается `socketio.AsyncServer` и `socketio.ASGIApp`
- События
  - `connect(sid, environ)` — регистрирует соединение
  - `disconnect(sid)` — удаляет соединение
  - `event(sid, data)` — точка входа для внешних сообщений вида
    `{ "event_type": str, "payload": dict, "trace_id"?: str, "user_id"?: str, "reply_event"?: str }`
    - формируется `Envelope` и публикуется в `bus`
    - ошибки отправляются как `error` событие клиенту
- Методы
  - `reply_to_sender(envelope)` — если пришло по WS и есть `sid`, отправит ответ этому клиенту
  - `emit(event_name, data, sid=None)` — отправка произвольного события(й)
  - `emit_envelope(envelope, sid=None)` — сериализует и отправляет как `event`
  - `connected_sids` — список текущих `sid`

SocketClient
- Назначение: простой клиент socket.io для подписки на события и отправки
- Конструктор: `SocketClient(url: str)`
- Методы
  - `on(event_type, callback)` — регистрирует колбек на события `event`
  - `connect()` — устанавливает соединение
  - `send(event_type, payload, trace_id=None)` — отправляет событие
  - `disconnect()` — закрывает соединение

Связанные разделы
- [Bus](bus.md)
- [События](events.md)
- [App](app.md)
