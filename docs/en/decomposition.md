# Block Decomposition Methodology

How to break a business task into blocks for khorsyio.

## Decomposition algorithm

### Step 1. Define the process boundaries

Input — what arrives from the user or trigger.
Output — what the final result should be.
Source — http, websocket, scheduler, or another handler.

### Step 2. Identify processing stages

Ask: what are the logically separate actions that happen between input and output? Each action that can be described in one sentence is a candidate for a block.

Signs that something is a separate block
- The action has its own responsibility (validation, calculation, check, persistence)
- The action can be replaced by a different implementation
- The action can be reused in another process
- The action accesses a separate resource (different table, external service)

Signs that actions belong in the same block
- The actions always execute together and make no sense individually
- There is no logical boundary between them
- Splitting them adds no clarity

### Step 3. Define structures

For each block define what it receives and what it returns. These are `msgspec.Struct`.

Two struct patterns

**Pipeline (shared state).** One struct is enriched as it passes through the chain. Each block adds its own fields.

```python
class OrderState(msgspec.Struct):
    # input data (filled by the first block)
    product: str = ""
    quantity: int = 0
    # data from PriceHandler
    unit_price: float = 0
    subtotal: float = 0
    # data from DiscountHandler
    discount_pct: float = 0
    total: float = 0
    # data from StockHandler
    stock_reserved: bool = False
    # pass-through status
    status: str = ""
    steps: list = msgspec.field(default_factory=list)
```

When to use: linear chains where each stage enriches the shared result. Convenient for tracing — the full path is visible in the final object.

**Transform (input/output).** Each block has its own input and output struct.

```python
class RawData(msgspec.Struct):
    records: list = msgspec.field(default_factory=list)

class Aggregated(msgspec.Struct):
    count: int = 0
    total: float = 0
    average: float = 0

class Formatted(msgspec.Struct):
    title: str = ""
    body: str = ""
```

When to use: stages with fundamentally different data. Input and output differ substantially. Blocks are more reusable.

### Step 4. Draw the event graph

```
input_event -> handler_1 -> event_a -> handler_2 -> event_b -> handler_3 -> output_event
```

Verify that
- Every published event has a subscriber
- There are no cycles (unless intentional)
- The final event is either caught by bus.request or handled by a terminal block

### Step 5. Define the namespace

If blocks belong to the same business process, group them in a Domain with a namespace.

```python
order = Domain()
order.namespace = "order"
order.handlers = [ValidateHandler, PriceHandler, ...]
```

Events become `order.validate`, `order.priced`, etc.

## Decomposition examples

### User registration

Task: user sends email and password, system registers them.

Stages
1. Validation (email format, password length)
2. Uniqueness check (email not taken)
3. Password hashing
4. Save to database
5. Send welcome email

```
user.validate -> user.check_unique -> user.hash_password -> user.save -> user.notify
```

Structures

```python
class RegisterIn(msgspec.Struct):
    email: str = ""
    password: str = ""

class UserState(msgspec.Struct):
    email: str = ""
    password: str = ""
    password_hash: str = ""
    user_id: int = 0
    status: str = ""
    steps: list = msgspec.field(default_factory=list)
```

Blocks 1–4 use the Pipeline pattern (shared UserState). Block 5 (notify) is terminal with `publishes=""` because no response is expected.

### Payment processing

Stages
1. Card validation
2. Limit check
3. Fraud check
4. Charge via gateway
5. Balance update
6. Receipt generation

```
payment.validate_card -> payment.check_limits -> payment.fraud_check -> payment.charge -> payment.update_balance -> payment.receipt
```

The fraud_check block can use conditional routing: if suspicious, it publishes `payment.review` instead of `payment.charge`.

### ETL pipeline (worker)

Scheduled task every hour. No HTTP.

```
etl.extract -> etl.transform -> etl.validate -> etl.load -> etl.notify
```

Blocks use the Transform pattern: Extract returns RawRows, Transform returns CleanRows, Validate returns ValidatedRows, Load returns LoadResult.

### Chat with moderation (WebSocket)

```
chat.message -> chat.moderate -> chat.enrich -> chat.broadcast
```

`moderate` checks for prohibited content. `enrich` adds timestamp and username. `broadcast` sends via transport.

## Rules for a good block

1. One block, one responsibility. If the block description contains "and", it probably needs to be split.

2. A block does not know who comes before or after it. It works only with its own input and context.

3. A block is idempotent where possible. Repeated calls with the same input produce the same result.

4. Errors are handled explicitly — either via status in the struct or via exception. Never swallowed silently.

5. A block logs its work via `ctx.trace_id`. This allows the full request path to be reconstructed.

6. Timeout matches the task. A fast calculation: 5s. An external API call: 30s. A heavy ETL: 120s.

7. Structs are defined before logic. Contract first, implementation second.

## Anti-patterns

**God handler** — a block that does everything. If `process` is longer than 30 lines, consider splitting.

**Hidden coupling** — a block directly imports and calls another block. Use events instead.

**Implicit contract** — a block accepts `dict` instead of a struct. Validation and documentation are lost.

**Global state** — a block writes to a global variable. Use structs or the database.

**Ignoring status** — a block does not check the status from the previous one. In a pipeline every block must verify that the previous stage succeeded.
