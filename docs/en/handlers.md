# Handlers and DI

See: [Documentation](README.md)

Base class
- Inherit from `khorsyio.core.handler.Handler`
- Contract fields
  - `subscribes_to: str` — event type to subscribe to
  - `publishes: str` — event type published as result (optional)
  - `input_type: type|None` — `msgspec.Struct` type for decoding input
  - `output_type: type|None` — declarative, for documentation and validation at project level
  - `timeout: float` — processing timeout in seconds
- Methods
  - `async def process(self, data, ctx) -> msgspec.Struct | Envelope | None` — implement in subclass
  - `async def handle(self, envelope: Envelope) -> Envelope | None` — implemented in base class, transforms data and wraps the response

DI in `__init__`
- When instantiated from a `Domain`, dependencies are injected by constructor parameter name
- Name mapping
  - `db -> app.db`
  - `client -> app.client`
  - `bus -> app.bus`
  - `transport -> app.transport`
  - `app -> app`
  - Any other name also receives `app`
- Helper class `Inject` is also available with names: `db`, `client`, `bus`, `transport`

Typical example
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
        # your logic; self._db and self._client are available
        return UserOut(id=1, name=data.name)
```

Processing flow
- Bus calls `handler.handle(envelope)`
- If `input_type` is set, input is decoded: `data = envelope.decode(input_type)`
- Return options
  - `None` — publish nothing
  - `Envelope` — return as is
  - `msgspec.Struct` with `publishes` set — wrapped into `Envelope` via `envelope.forward(...)`

Related sections
- [Bus](bus.md)
- [Events](events.md)
- [Domains and DI](domain.md)
