**Khorsyio Framework**

Applicability Analysis and Architecture Patterns

Version 2.0  |  2026

# 1. Core framework concepts

Khorsyio is an async Python framework with an event-driven architecture. Business logic is implemented through isolated blocks (handlers) that interact exclusively through typed events on the bus. There are no direct calls between blocks. Stack: asyncio, msgspec (serialization), asyncpg (postgres), socketio (websocket), ASGI (http).

| Concept | Description |
| :---- | :---- |
| Handler | Unit of business logic. One input (subscribes_to), one output (publishes), isolated state. Implements a single method process(data, ctx) |
| Struct (msgspec) | Typed data contract between blocks. Serves as documentation, validation, and serialization simultaneously. 10–50x faster than pydantic |
| Event | A string of the form namespace.action (e.g. order.validate). The bus routes by subscriptions automatically |
| Context (ctx) | Request-wide context: trace_id, user_id, extra (dict). Automatically propagated through the entire chain when forwarding between handlers |
| Domain | Grouping of handlers and routes under a shared namespace. With namespace='order', event 'validate' becomes 'order.validate' |
| Bus | Central bus. Routing, bus.request (publish + wait), per-handler metrics, event log, scheduled tasks, graceful shutdown with drain |
| Error | Standard error structure: code, message, source, trace_id, details. Envelope.is_error for checking |
| DI | Dependency injection by parameter name in __init__: db, client, bus, transport, app. No decorators or configuration |

## Block anatomy

Every handler defines four attributes and implements a single `process` method. Input deserialization, output serialization, envelope creation, and context propagation are handled automatically by the framework.

```python
class PriceHandler(Handler):
    subscribes_to = 'validated'      # input event
    publishes = 'priced'             # output event
    input_type = OrderState          # input struct
    output_type = OrderState         # output struct
    timeout = 5.0                    # processing timeout

    async def process(self, data: OrderState, ctx: Context) -> OrderState:
        data.unit_price = CATALOG[data.product]
        data.total = data.unit_price * data.quantity
        data.status = 'priced'
        return data
```

## Dependency injection

The parameter name in `__init__` determines what is injected. The framework automatically passes the right component when creating a handler through Domain. Mapping: `db -> app.db` (asyncpg pool), `client -> app.client` (httpx session), `bus -> app.bus`, `transport -> app.transport` (socketio), `app -> the full application object`. Multiple dependencies can be combined.

```python
class FetchHandler(Handler):
    def __init__(self, db, client):     # receives app.db and app.client
        self._db = db
        self._client = client
```

## Two data structure patterns

**Pipeline (shared state).** One struct is enriched as it passes through the chain. Each block adds its own fields without touching others. Useful for tracing — the final object shows the full path. Used in linear business processes (order, ticket, airdrop).

**Transform (input/output).** Each block has its own input and output struct. Input and output differ substantially by nature. Blocks are more reusable. Used in ETL, report pipelines, and data processing.

# 2. Framework operating modes

The framework supports five data flow modes. Modes can be combined: one process can start as an HTTP request (mode 1), include conditional routing (mode 5), and end with fan-out notifications (mode 4).

## Mode 1. HTTP -> chain -> response to client

An HTTP endpoint starts the chain via `bus.request` and synchronously waits for the final response with a matching `trace_id`. The client receives the result of the entire chain in a single HTTP response. Suitable for REST APIs where the client expects a result.

| # | Event | Description |
| :---- | :---- | :---- |
| 1 | POST /order | Client sends request with JSON body |
| 2 | order.validate | Field validation, data normalization |
| 3 | order.priced | Price calculation from catalog |
| 4 | order.discounted | Loyalty discount applied by customer_id |
| 5 | order.reserved | Stock check, warehouse reservation |
| 6 | order.paid | Charge, payment reference generated |
| 7 | order.confirmed | order_id generated, final status |
| 8 | HTTP 201 | bus.request receives response, Response.json to client |

**Key mechanism:** `bus.request` publishes the first event in the chain and creates a waiter keyed on `response_type + trace_id`. When the final handler publishes an event with the same `trace_id`, the waiter fires and returns the envelope to the calling code. On timeout, an error envelope with `code='timeout'` is returned.

```python
result = await bus.request('order.validate', data,
    response_type='order.confirmed', source='http', timeout=10.0)
```

## Mode 2. Worker / Fire-and-forget

A scheduled task or publish without waiting. The chain runs autonomously; the result is written to a database, log, or external service. No `bus.request`, no HTTP, no waiting for a response.

| # | Event | Description |
| :---- | :---- | :---- |
| 1 | schedule / publish | Scheduler or external trigger starts the chain |
| 2 | report.collect | Collect metrics from multiple sources |
| 3 | report.aggregated | Aggregation: count, total, average, max, min |
| 4 | report.enriched | Add metadata: period, version, source |
| 5 | report.formatted | Format into a text report |
| 6 | report.delivered | Send: log, webhook, email, S3 file |

```python
app.bus.schedule('report.collect', ReportTrigger(), interval=3600.0)
await bus.publish('report.collect', ReportTrigger(), source='manual')
```

## Mode 3. WebSocket -> chain -> reply to sender

A WebSocket client sends an event. The chain processes the data. The final handler returns the response to the specific client via `transport.reply_to_sender(envelope)`. The client's `sid` is stored in `ctx.extra['_ws_sid']` automatically and propagated through the entire chain.

| # | Event | Description |
| :---- | :---- | :---- |
| 1 | ws: auction.bid | Participant sends a bid via WebSocket |
| 2 | bid.validate | Check: lot is active, amount exceeds current, balance sufficient |
| 3 | bid.lock | Optimistic lock for concurrent bids |
| 4 | bid.persist | Write bid to DB, update current leader |
| 5 | bid.notify_all | Fan-out: broadcast to all connected via transport.emit |
| 6 | bid.reply_sender | Reply to initiator via transport.reply_to_sender |

## Mode 4. Fan-out (parallel subscribers)

Multiple handlers subscribe to the same event. All receive a copy and work in parallel via `asyncio.gather`. Failure of one does not block the others. Adding a new handler requires no changes to existing blocks and no redeployment of other parts of the system.

| # | Event | Description |
| :---- | :---- | :---- |
| 1 | order.confirmed | Fan-out starting point (final order event) |
| 2 | SendEmailHandler | Send confirmation to client |
| 3 | UpdateInventoryHandler | Deduct items from warehouse stock |
| 4 | CreateInvoiceHandler | Generate invoice |
| 5 | TrackAnalyticsHandler | Record metrics |
| 6 | NotifySlackHandler | Notify the team |

**Why this works:** in the classic approach, adding a new side effect requires finding the right place in the code and inserting a call, risking breaking the existing flow. In the event-driven model, a new handler simply subscribes to the event. Existing code is untouched. Testing is isolated.

## Mode 5. Conditional routing

A handler decides which event to publish, returning an Envelope directly. `publishes=''`. Enables flow branching without an external orchestrator.

| # | Event | Description |
| :---- | :---- | :---- |
| 1 | payment.process | RouterHandler receives payment |
| 2 | amount > 10000 | payment.manual_review (manual check) |
| 3 | currency != EUR | payment.convert (currency conversion) |
| 4 | 3ds required | payment.verify_3ds (confirmation) |
| 5 | standard path | payment.execute (execution) |

```python
async def process(self, data, ctx):
    if data.amount > 10000:
        return Envelope.create('payment.manual_review', data,
            source='RouterHandler', trace_id=ctx.trace_id)
```

**Important:** when creating an Envelope manually, always pass `trace_id` from `ctx` to preserve end-to-end tracing. Alternatively, use `envelope.forward()` which does this automatically.

# 3. Complex application scenarios

Detailed cases where the event-driven block architecture provides the greatest advantage. Each shows the flow, why khorsyio wins, and what would be harder without the framework.

## Scenario A. Airdrop campaign with multi-chain distribution

A typical blockchain task: accept a claim, check eligibility, determine the network, sign the transaction, send it, and monitor status. Each stage is isolated with a clear contract.

| # | Event | Description |
| :---- | :---- | :---- |
| 1 | airdrop.submit | Accept claim: wallet address, network, campaign_id |
| 2 | airdrop.verify | Check eligibility: whitelist, balance, previous claims |
| 3 | airdrop.route | Conditional routing: base / polygon / bsc / arbitrum |
| 4 | airdrop.sign | Generate EIP-712 signature for the specific network |
| 5 | airdrop.broadcast | Send transaction via RPC provider |
| 6 | airdrop.monitor | Monitor status: pending -> confirmed / failed |
| 7 | airdrop.complete | Fan-out: update DB + notify user + analytics |

### Why khorsyio wins here

**RPC call isolation.** Each network (base, polygon, bsc) is implemented as a separate handler with its own timeout and error handling. An RPC error on one network does not affect processing on others. In monolithic code this is typically a large switch-case with try-except where an error in one branch can block the entire flow.

**Conditional routing at the route step.** The `airdrop.route` handler analyzes the network from the claim and publishes to the appropriate branch: `airdrop.sign.base`, `airdrop.sign.polygon`, etc. Each branch can have its own signing logic. Adding a new network is a new handler, not a change to existing code.

**Fan-out at the end.** After transaction confirmation, three actions run in parallel: updating the DB, sending a notification, recording analytics. If the email service is down, balances in the DB are still updated. In synchronous code, a failed email send often breaks the entire processing flow.

**Tracing.** A single `trace_id` travels through the entire chain from submit to complete. When troubleshooting a specific airdrop, you can query `event_log` by `trace_id` and see exactly which step failed and with what error. In regular code this requires manually threading a `correlation_id` through every call.

## Scenario B. Real-time auction via WebSocket

An auction platform with real-time bidding. Requires a synchronous response to the participant and parallel notification of all observers. A combination of modes 3, 4, and 2.

| # | Event | Description |
| :---- | :---- | :---- |
| 1 | ws: auction.bid | Participant sends a bid via WebSocket |
| 2 | bid.validate | Check: lot active, amount exceeds current, balance sufficient |
| 3 | bid.lock | Optimistic lock for concurrent bids |
| 4 | bid.persist | Write bid to DB, update current leader |
| 5 | bid.notify_all | Fan-out: broadcast to all via transport.emit |
| 6 | bid.reply_sender | Reply to initiator via transport.reply_to_sender |
| 7 | bid.check_end | Worker: background check whether auction time has expired |

### Why khorsyio wins here

**Natural fit with the WebSocket model.** An event from a WS client lands on the bus as an envelope with `ctx.extra['_ws_sid']`. This is not a REST-to-realtime adaptation; it is a native event-driven approach. The bus processes WS events the same way it processes HTTP events — with the same handlers, metrics, and tracing.

**Separation of response and notification.** `bid.reply_sender` sends a personal response to the specific participant (accepted/rejected). `bid.notify_all` broadcasts an update to all observers. These are two separate handlers, two separate concerns. In typical WS code both actions are usually in one function, and adding notification logic complicates bid processing.

**Concurrent bids.** `bid.lock` implements optimistic locking as an isolated block. If two bids arrive simultaneously, the bus processes them sequentially (asyncio single-threaded). The lock handler checks the current price in the DB and rejects the losing bid. Locking logic is decoupled from validation and persistence logic.

## Scenario C. Ticket system with automatic classification

A real example from the demo: a ticket passes through 4 blocks, each enriching a shared `TicketState`. Keyword-based classification, team assignment, postgres persistence.

| # | Event | Description |
| :---- | :---- | :---- |
| 1 | ticket.validate | Validation: title >= 3 chars, author >= 2. Normalization |
| 2 | ticket.classified | Text analysis: determine category and priority by keywords |
| 3 | ticket.assigned | Assign team by category: billing->finance, technical->engineering |
| 4 | ticket.created | INSERT to postgres, return id and timestamp |

### Why khorsyio wins here

**Replacing one block without side effects.** ClassifyHandler currently uses keyword matching. Tomorrow it can be replaced with an ML classifier. The contract does not change: input is TicketState with `status='validated'`, output is TicketState with `category` and `priority`. The other blocks do not know about the replacement.

**Extending without rework.** Need to add SLA calculation? A new handler `ticket.sla_calculated` between assigned and created. Need Slack notifications for critical tickets? A new handler subscribed to `ticket.created` checking priority. Not a single line of existing code changes.

**Shared TicketState as audit trail.** The `steps` field collects each block's action: `['validate:ok', 'classify:technical/critical', 'assign:engineering_team', 'persist:id=42']`. A single response shows the full ticket path. In the classic approach, an equivalent audit trail requires a separate logging mechanism.

## Scenario D. DeFi: swap transaction processing

| # | Event | Description |
| :---- | :---- | :---- |
| 1 | swap.init | Receive parameters: token_in, token_out, amount, slippage |
| 2 | swap.quote | Request quote via external aggregator API |
| 3 | swap.validate_slippage | Check slippage is within acceptable range |
| 4 | swap.approve_check | Check token approval for the contract |
| 5 | swap.build_tx | Build transaction with gas parameters |
| 6 | swap.sign_submit | Sign and broadcast to network |
| 7 | swap.monitor | Worker: poll status every N seconds |
| 8 | swap.finalize | Fan-out: update balances + history + notification |

### Why khorsyio wins here

**Precise error localization.** A swap has many failure points: aggregator API unavailable (swap.quote), slippage exceeded (swap.validate_slippage), gas spiked (swap.build_tx), transaction rejected (swap.sign_submit). Each error is localized in a specific handler with a specific `error.code` and `error.source`. In a monolithic 200-line function, a 'transaction failed' error does not tell you at which exact stage the failure occurred.

**Timeout per handler.** `swap.quote` has `timeout=5s` (external API). `swap.monitor` has `timeout=120s` (waiting for network confirmation). `swap.validate_slippage` has `timeout=1s` (pure calculation). Each block gets a timeout appropriate to its task. With a shared try-except there is one timeout for everything.

**Per-handler metrics.** `bus.metrics.snapshot()` will show that `swap.quote` averages 800ms and `swap.build_tx` 50ms. If `swap.quote` average climbs from 800ms to 3s, the problem is with the aggregator API. Without per-handler metrics, only total swap time is visible, and the bottleneck is unclear.

## Scenario E. ETL pipeline with data transformation

| # | Event | Description |
| :---- | :---- | :---- |
| 1 | etl.ingest | Read raw data from source (S3, API, DB) |
| 2 | etl.clean | Remove null records, normalize types, deduplicate |
| 3 | etl.validate | Check business rules, flag invalid records |
| 4 | etl.enrich | Supplement with reference data |
| 5 | etl.transform | Apply business transformations, calculate aggregates |
| 6 | etl.load | Load into the target storage |
| 7 | etl.report | Generate report: processed, errors, skipped |

Status-based error strategy via struct: each block checks the previous block's status and if it does not match, passes the data forward without processing. The final `etl.report` receives a state with full information about what happened at each stage.

# 4. Error handling strategies

The framework provides two approaches to errors. The choice determines the behavior of the entire chain.

## Strategy 1. Status in struct (soft mode)

The block writes an error status into the data and returns it. Subsequent blocks check the status and skip processing. Data travels the full chain, collecting information along the way.

```python
if not data.wallet_address:
    data.status = 'error: wallet_address required'
    return data  # continue through the chain without processing
```

**When to use:** partial errors are acceptable, errors need to be accumulated across multiple fields, it is important to preserve an audit trail even for invalid data, the client should receive a detailed response about the reason for failure.

## Strategy 2. Exception (hard mode)

The block raises an exception; the bus catches it and creates an error envelope with `code='handler_error'`. The chain stops. `bus.request` receives the error envelope.

```python
if not await self._has_permission(ctx.user_id):
    raise PermissionError('access denied')
```

**When to use:** the error is critical and further processing is meaningless, a business logic invariant is violated, a system error occurs (DB unavailable, external service not responding).

## Combined strategy

In practice, a single chain uses both approaches: validation errors via status (user left a field empty), system errors via exception (RPC connection dropped). `bus.request` handles both correctly: a status error arrives in the final envelope, an exception arrives as an error envelope with `is_error=True`.

# 5. Applicability matrix

## High applicability

| Scenario | Rationale | Rating |
| :---- | :---- | :---- |
| Multi-step business processes (order, payment, onboarding) | Natural decomposition into stages with their own contracts. Each stage is tested in isolation. Replacing one block does not affect others | Excellent |
| Real-time systems (auctions, chats, trading) | Event model natively matches WebSocket. WS sid automatically in ctx. reply_to_sender for personal responses | Excellent |
| Blockchain/DeFi operations | Isolated RPC calls, conditional routing by network, fan-out at the end, per-handler timeout for different services | Excellent |
| Fan-out reactions | Add a new side effect without changing existing code. Fault tolerance: one handler failing does not block others | Excellent |
| Workers / Scheduled tasks | Fire-and-forget chains, built-in scheduler, metrics and event log for monitoring autonomous processes | Excellent |
| Integration layers | External API isolated in a handler with its own timeout and retry. Unified error pattern for all integrations | Good |
| ETL pipelines | Shared state across stages. Status-based error handling. Final report shows per-stage statistics | Good |
| Systems with audit trail | trace_id is end-to-end, steps in struct, event_log built in, per-handler metrics. No separate infrastructure needed | Good |

## Framework overhead

| Scenario | Rationale |
| :---- | :---- |
| Simple CRUD without business logic | Accept request, write to DB, return response. The event layer adds no value. A direct call is sufficient |
| Single-step transformations | No chain, no reason for a bus. A plain async function is simpler |
| Admin scripts | Migrations, one-off imports. Event-driven architecture solves a non-existent problem |
| Static API endpoints (health, metadata) | Responses without processing. Overhead with no benefit. In khorsyio these endpoints are wired directly via router.get without the bus |

## Fundamental limitations

| Scenario | Rationale |
| :---- | :---- |
| Distributed systems | The bus operates in-process (asyncio.Queue). For inter-service communication you need Kafka, RabbitMQ, or NATS. Khorsyio solves the problem within a single process |
| CPU-intensive computation | ML inference, crypto mining, rendering. The bottleneck is the CPU, not code organization. The event bus adds overhead without benefit |
| Microservices with network transport | The framework does not provide transport between services. Each service can use khorsyio internally, but communication between services is out of scope |

# 6. Monitoring and observability

Built-in tools with no additional dependencies.

| Tool | What it provides |
| :---- | :---- |
| bus.metrics.snapshot() | Per-handler: processed (count), errors, avg_ms (average time), last_error |
| bus.event_log.recent(N) | Last N events in a ring buffer. Filter by event_type and/or trace_id. Buffer size is configurable |
| trace_id | End-to-end identifier. Propagated automatically through forward. Allows reconstructing the full request path |
| ctx.source | Request source (http, ws:sid, scheduler) available in every handler |
| Endpoint /metrics | JSON with metrics for all handlers. Added in one line |
| Endpoint /events | JSON with event log. Supports query parameters n, type, trace |

```python
# metrics endpoint
app.router.get('/metrics', lambda req, send: Response.json(send, app.bus.metrics.snapshot()))

# view events for a specific request
app.bus.event_log.recent(20, trace_id='abc123')
```

# 7. Graceful shutdown

When the application stops, the bus performs a controlled shutdown.

| Stage | Action |
| :---- | :---- |
| 1. Stop accepting | Flag _running=False, new events are not accepted |
| 2. Cancel scheduled | All scheduled tasks are cancelled via task.cancel() |
| 3. Drain queue | Remaining events in the queue are processed within drain_timeout (default 5s) |
| 4. Resolve waiters | Pending bus.request calls receive an error envelope with code='shutdown' |
| 5. Close resources | Close HTTP client, database pool, WebSocket connections |

# 8. Pre-development checklist

Every handler must pass this checklist before writing any code.

| Item | What to define |
| :---- | :---- |
| Input struct | msgspec.Struct with typed fields and defaults. This is the contract with the previous block |
| Output struct | msgspec.Struct. Contract with the next block. For the pipeline pattern — the same struct |
| subscribes_to | Event to subscribe to. With Domain namespace the prefix is added automatically |
| publishes | Event to publish. Empty string for manual control (conditional routing) or terminal block (fan-out receiver) |
| Dependencies | Which components are needed: db, client, bus, transport. Defined via parameter name in __init__ |
| timeout | Maximum processing time. Appropriate to the task: calculation 1–5s, external API 15–30s, tx monitoring up to 120s |
| Error strategy | Status in struct (soft, data continues through) or exception (hard, chain stops) |
| Namespace | Confirm the handler is in Domain.handlers and namespace is correct |
