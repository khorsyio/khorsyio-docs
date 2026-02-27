# Обработчики (Handlers) и DI

См. оглавление: [Документация](README.md)

Базовый класс
- Класс наследуется от `khorsyio.core.handler.Handler`
- Поля контракта
  - `subscribes_to: str` — тип события, на который подписан
  - `publishes: str` — тип события, которое публикуется как результат (опционально)
  - `input_type: type|None` — тип структуры `msgspec.Struct` для декодирования входа
  - `output_type: type|None` — декларативно, для информации/валидации на уровне проекта
  - `timeout: float` — таймаут обработки (секунды)
- Методы
  - `async def process(self, data, ctx) -> msgspec.Struct | Envelope | None` — реализовать в наследнике
  - `async def handle(self, envelope: Envelope) -> Envelope | None` — реализовано в базовом, преобразует данные и оборачивает ответ

DI в __init__
- При создании экземпляра из `Domain` зависимости пробрасываются по имени параметра конструктора
- Отображение имен
  - `db -> app.db`
  - `client -> app.client`
  - `bus -> app.bus`
  - `transport -> app.transport`
  - `app -> app`
  - Любое иное имя также получит `app`
- Также доступен вспомогательный класс `Inject` с именами: `db`, `client`, `bus`, `transport`

Типовой пример
```python
from khorsyio import Handler, Context
import msgspec

class UserIn(msgspec.Struct):
    name: str

class UserOut(msgspec.Struct):
    id: int
    name: str

class CreateUser(Handler):
    subscribes_to = "user.create"
    publishes = "user.created"
    input_type = UserIn
    output_type = UserOut

    def __init__(self, db, client):
        self._db = db
        self._client = client

    async def process(self, data: UserIn, ctx: Context) -> UserOut:
        # ваша логика, можно использовать self._db и self._client
        return UserOut(id=1, name=data.name)
```

Поток обработки
- Шина вызывает `handler.handle(envelope)`
- Если задан `input_type`, вход декодируется: `data = envelope.decode(input_type)`
- Возврат вариантов
  - `None` — ничего не публиковать
  - `Envelope` — вернуть как есть
  - `msgspec.Struct` при наличии `publishes` — будет обернут в `Envelope` через `envelope.forward(...)`

Связанные разделы
- [Bus](bus.md)
- [События](events.md)
- [Домены и DI](domain.md)
