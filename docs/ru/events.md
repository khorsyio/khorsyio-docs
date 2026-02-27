# События: Envelope, Context, Error

См. оглавление: [Документация](README.md)

Context
- Поля
  - `trace_id` — генерируется автоматически, сквозной идентификатор
  - `timestamp` — время создания
  - `source` — источник события (например `api` или `ws:<sid>`)
  - `user_id` — идентификатор пользователя
  - `extra` — произвольные данные

Error
- Поля
  - `code` — строковый код ошибки
  - `message` — описание
  - `source` — источник сформировавший ошибку
  - `trace_id` — для связывания с исходным запросом
  - `details` — расширенные данные

Envelope
- Поля
  - `ctx: Context`
  - `event_type: str`
  - `payload: bytes` — сериализованные через msgspec данные
  - `error: Error | None`
- Методы/свойства
  - `create(event_type, data, source="", trace_id=None, user_id="", extra=None)` — собрать новый конверт
  - `error_from(original, message, code="error", source="", details=None)` — создать ошибочный ответ
  - `decode(typ)` — декодировать `payload` в тип `typ` через msgspec
  - `forward(event_type, data, source="")` — переслать новый конверт с тем же trace_id/user_id/extra
  - `trace_id` — свойство для быстрого доступа
  - `is_error` — признак ошибки

Использование
- Для публикации используйте `bus.publish("type", data_struct)` — конверт будет собран автоматически
- Для RPC-подобных вызовов используйте `bus.request(...)` — вернет `Envelope`
- Проверяйте `resp.is_error` и поля `resp.error.*`

Связанные разделы
- [Шина событий](bus.md)
- [Обработчики](handlers.md)
