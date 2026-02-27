# Block Development Templates

Ready-made recipes for common tasks.

## Template: CRUD domain

```python
import msgspec
from khorsyio import Domain, Handler, Response, Route, Context

# --- structures ---

class ItemIn(msgspec.Struct):
    name: str = ""
    value: float = 0

class Item(msgspec.Struct):
    id: int = 0
    name: str = ""
    value: float = 0

class ItemList(msgspec.Struct):
    items: list = msgspec.field(default_factory=list)
    total: int = 0

class ItemDeleted(msgspec.Struct):
    id: int = 0
    ok: bool = False

# --- handlers ---

class CreateItem(Handler):
    subscribes_to = "create"
    publishes = "created"
    input_type = ItemIn
    output_type = Item

    def __init__(self, db):
        self._db = db

    async def process(self, data: ItemIn, ctx: Context) -> Item:
        row = await self._db.fetchrow(
            "insert into items (name, value) values ($1, $2) returning id, name, value",
            data.name, data.value)
        return Item(**row)


class GetItem(Handler):
    subscribes_to = "get"
    publishes = "got"
    input_type = Item  # only id is used
    output_type = Item

    def __init__(self, db):
        self._db = db

    async def process(self, data: Item, ctx: Context) -> Item:
        row = await self._db.fetchrow("select id, name, value from items where id = $1", data.id)
        if not row:
            raise ValueError(f"item {data.id} not found")
        return Item(**row)


class ListItems(Handler):
    subscribes_to = "list"
    publishes = "listed"
    input_type = ItemList
    output_type = ItemList

    def __init__(self, db):
        self._db = db

    async def process(self, data, ctx: Context) -> ItemList:
        rows = await self._db.fetch("select id, name, value from items order by id")
        return ItemList(items=[dict(r) for r in rows], total=len(rows))


class DeleteItem(Handler):
    subscribes_to = "delete"
    publishes = "deleted"
    input_type = Item
    output_type = ItemDeleted

    def __init__(self, db):
        self._db = db

    async def process(self, data: Item, ctx: Context) -> ItemDeleted:
        result = await self._db.execute("delete from items where id = $1", data.id)
        return ItemDeleted(id=data.id, ok="DELETE 1" in result)


# --- routes ---

async def route_create(req, send):
    data = await req.json(ItemIn)
    result = await req.state["bus"].request("item.create", data, response_type="item.created", timeout=5)
    if result.is_error:
        await Response.error(send, result.error.message, 500)
        return
    await Response.json(send, result.decode(Item), status=201)


async def route_get(req, send):
    id = int(req.path_params["id"])
    result = await req.state["bus"].request("item.get", Item(id=id), response_type="item.got", timeout=5)
    if result.is_error:
        await Response.error(send, result.error.message, 404)
        return
    await Response.json(send, result.decode(Item))


async def route_list(req, send):
    result = await req.state["bus"].request("item.list", ItemList(), response_type="item.listed", timeout=5)
    await Response.json(send, result.decode(ItemList))


async def route_delete(req, send):
    id = int(req.path_params["id"])
    result = await req.state["bus"].request("item.delete", Item(id=id), response_type="item.deleted", timeout=5)
    await Response.json(send, result.decode(ItemDeleted))


# --- domain ---

items = Domain()
items.namespace = "item"
items.handlers = [CreateItem, GetItem, ListItems, DeleteItem]
items.routes = [
    Route("POST", "/items", route_create),
    Route("GET", "/items", route_list),
    Route("GET", "/items/{id}", route_get),
    Route("DELETE", "/items/{id}", route_delete),
]
```

## Template: pipeline with shared state

```python
class PipelineState(msgspec.Struct):
    # input data
    input_field: str = ""
    # each handler adds its own fields
    step1_result: str = ""
    step2_result: float = 0
    step3_result: bool = False
    # tracing
    status: str = ""
    steps: list = msgspec.field(default_factory=list)


class Step1(Handler):
    subscribes_to = "step1"
    publishes = "step1_done"
    input_type = PipelineState
    output_type = PipelineState

    async def process(self, data, ctx):
        data.step1_result = f"processed: {data.input_field}"
        data.status = "step1_done"
        data.steps = data.steps + ["step1:ok"]
        return data


class Step2(Handler):
    subscribes_to = "step1_done"
    publishes = "step2_done"
    input_type = PipelineState
    output_type = PipelineState

    async def process(self, data, ctx):
        if data.status != "step1_done":
            return data  # skip on error
        data.step2_result = 42.0
        data.status = "step2_done"
        data.steps = data.steps + ["step2:ok"]
        return data
```

## Template: transform chain

```python
class RawInput(msgspec.Struct):
    data: list = msgspec.field(default_factory=list)

class Cleaned(msgspec.Struct):
    records: list = msgspec.field(default_factory=list)
    removed: int = 0

class Enriched(msgspec.Struct):
    records: list = msgspec.field(default_factory=list)
    metadata: dict = msgspec.field(default_factory=dict)

class Output(msgspec.Struct):
    summary: str = ""
    count: int = 0


class CleanHandler(Handler):
    subscribes_to = "clean"
    publishes = "cleaned"
    input_type = RawInput
    output_type = Cleaned

    async def process(self, data, ctx):
        valid = [r for r in data.data if r.get("value") is not None]
        return Cleaned(records=valid, removed=len(data.data) - len(valid))


class EnrichHandler(Handler):
    subscribes_to = "cleaned"
    publishes = "enriched"
    input_type = Cleaned
    output_type = Enriched

    async def process(self, data, ctx):
        return Enriched(records=data.records, metadata={"source": ctx.source, "count": len(data.records)})
```

## Template: fan-out (multiple subscribers)

```python
# three handlers subscribe to the same event
# all receive a copy and work in parallel

class SendEmail(Handler):
    subscribes_to = "user.registered"
    publishes = ""
    input_type = UserRegistered

    def __init__(self, client):
        self._client = client

    async def process(self, data, ctx):
        await self._client.post("https://email-api/send", json_data={"to": data.email, "template": "welcome"})


class UpdateCRM(Handler):
    subscribes_to = "user.registered"
    publishes = ""
    input_type = UserRegistered

    def __init__(self, db):
        self._db = db

    async def process(self, data, ctx):
        await self._db.execute("insert into crm_contacts (email) values ($1)", data.email)


class TrackAnalytics(Handler):
    subscribes_to = "user.registered"
    publishes = ""
    input_type = UserRegistered

    async def process(self, data, ctx):
        pass  # send to analytics
```

## Template: conditional routing

```python
from khorsyio import Envelope

class RouterHandler(Handler):
    subscribes_to = "payment.process"
    publishes = ""  # manual control
    input_type = PaymentIn

    async def process(self, data, ctx):
        if data.amount > 10000:
            return Envelope.create("payment.manual_review", data,
                                   source="RouterHandler", trace_id=ctx.trace_id)
        if data.currency != "EUR":
            return Envelope.create("payment.convert", data,
                                   source="RouterHandler", trace_id=ctx.trace_id)
        return Envelope.create("payment.execute", data,
                               source="RouterHandler", trace_id=ctx.trace_id)
```

## Template: scheduled worker

```python
class CleanupTrigger(msgspec.Struct):
    max_age_days: int = 30

class CleanupResult(msgspec.Struct):
    deleted: int = 0

class CleanupHandler(Handler):
    subscribes_to = "cleanup"
    publishes = "cleanup_done"
    input_type = CleanupTrigger
    output_type = CleanupResult

    def __init__(self, db):
        self._db = db

    async def process(self, data, ctx):
        result = await self._db.execute(
            "delete from logs where created_at < now() - interval '$1 days'", data.max_age_days)
        count = int(result.split()[-1]) if result else 0
        return CleanupResult(deleted=count)


# in app.py
app.bus.schedule("maintenance.cleanup", CleanupTrigger(max_age_days=30), interval=3600)
```

## Template: external API integration

```python
class ExternalRequest(msgspec.Struct):
    endpoint: str = ""
    params: dict = msgspec.field(default_factory=dict)

class ExternalResponse(msgspec.Struct):
    status: int = 0
    data: dict = msgspec.field(default_factory=dict)
    ok: bool = False

class ExternalApiHandler(Handler):
    subscribes_to = "external.call"
    publishes = "external.result"
    input_type = ExternalRequest
    output_type = ExternalResponse
    timeout = 30.0

    def __init__(self, client):
        self._client = client

    async def process(self, data, ctx):
        try:
            resp = await self._client.get(data.endpoint, timeout=15.0)
            import msgspec
            body = msgspec.json.decode(resp["body"].encode())
            return ExternalResponse(status=resp["status"], data=body, ok=resp["status"] < 400)
        except Exception as e:
            return ExternalResponse(status=0, data={"error": str(e)}, ok=False)
```

## Template: auth middleware and ctx

```python
# middleware extracts user_id
async def auth_middleware(req):
    token = req.header("authorization")
    if token:
        req.state["user_id"] = validate_token(token)

app.router.use(auth_middleware)

# route passes user_id to bus.request
async def my_route(req, send):
    result = await req.state["bus"].request(
        "domain.action", data,
        response_type="domain.result",
        user_id=req.state.get("user_id", ""))

# handler checks ctx.user_id
class SecureHandler(Handler):
    async def process(self, data, ctx):
        if not ctx.user_id:
            raise PermissionError("auth required")
        # ...
```

## Pre-development checklist

1. Input struct is defined (msgspec.Struct)
2. Output struct is defined
3. Input event is defined (subscribes_to)
4. Output event is defined (publishes)
5. Dependencies are defined (db, client, bus, transport, or none)
6. Timeout is defined
7. Error strategy is defined (status in struct or exception)
8. Domain namespace is defined
