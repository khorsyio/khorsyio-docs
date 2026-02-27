# Шина событий (Bus) и журнал событий

См. оглавление: [Документация](README.md)

Назначение
- Внутренняя асинхронная шина для доставки событий (`Envelope`) обработчикам (`Handler`)
- Поддержка публикации/запросов, таймаутов, планировщика, метрик и журнала событий

Ключевые возможности
- `register(handler)` — подписывает обработчик на `handler.subscribes_to`
- `publish(event_or_envelope, data=None, source="", trace_id=None, user_id="", extra=None)`
  - Принимает строковый тип события и данные `msgspec.Struct` или готовый `Envelope`
- `request(event_type, data, response_type=None, source="", timeout=None, user_id="", extra=None)`
  - Ждет первое подходящее событие-ответ
  - Если `response_type` не указан, ожидается `{event_type}.done`
- `schedule(event_type, data, interval, source="scheduler")` — периодическая публикация
- `start()` — запуск обработчика очереди и планировщика
- `stop(drain_timeout=5.0)` — мягкая остановка с дожиданием очереди
- `on_error(callback)` — колбек на ошибки доставки/обработки
- `validate_graph()` — предупреждения по графу (отсутствующие подписчики и т.п.)

Метрики и журнал
- `bus.metrics.snapshot()` — статистика по хендлерам: processed, errors, avg_ms
- `bus.event_log.recent(n=50, event_type=None, trace_id=None)` — последние записи
- `bus.event_log.snapshot()` — весь текущий буфер

Таймауты
- Таймаут по умолчанию для хендлеров берется из `settings.bus.handler_timeout`
- Можно переопределить в поле `Handler.timeout`

Ошибки
- Если `process` бросает исключение, в лог пишется ошибка, метрики фиксируют неуспех
- Для `request` по таймауту возбуждается `TimeoutError`

Пример публикации
```python
from khorsyio import Bus, Envelope
import msgspec

class Ping(msgspec.Struct):
    x: int

# Публикация с автосборкой конверта
await app.bus.publish("ping", Ping(x=1), source="api")
```

Пример запроса/ответа
```python
# Запрос
resp = await app.bus.request("user.create", UserIn(name="A"), response_type="user.created")
if resp.is_error:
    # см. [events.md](events.md)
    ...
```

Связанные разделы
- [События: Envelope, Context, Error](events.md)
- [Обработчики](handlers.md)
- [App](app.md)
