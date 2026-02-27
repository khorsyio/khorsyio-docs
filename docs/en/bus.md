# Event Bus and Event Log

See: [Documentation](README.md)

Purpose
- Internal async bus for delivering events (`Envelope`) to handlers (`Handler`)
- Supports publish/request, timeouts, scheduler, metrics, and event log

Key capabilities
- `register(handler)` — subscribes a handler to `handler.subscribes_to`
- `publish(event_or_envelope, data=None, source="", trace_id=None, user_id="", extra=None)`
  - Accepts a string event type and `msgspec.Struct` data, or a ready-made `Envelope`
- `request(event_type, data, response_type=None, source="", timeout=None, user_id="", extra=None)`
  - Waits for the first matching response event
  - If `response_type` is not specified, `{event_type}.done` is expected
- `schedule(event_type, data, interval, source="scheduler")` — periodic publishing
- `start()` — starts the queue processor and scheduler
- `stop(drain_timeout=5.0)` — graceful stop, waits for the queue to drain
- `on_error(callback)` — callback for delivery and processing errors
- `validate_graph()` — warnings about the handler graph (missing subscribers, etc.)

Metrics and event log
- `bus.metrics.snapshot()` — per-handler stats: processed, errors, avg_ms
- `bus.event_log.recent(n=50, event_type=None, trace_id=None)` — recent entries
- `bus.event_log.snapshot()` — full current buffer

Timeouts
- Default handler timeout is taken from `settings.bus.handler_timeout`
- Can be overridden per handler via `Handler.timeout`

Errors
- If `process` raises an exception, the error is written to the log and metrics record the failure
- For `request`, a timeout raises `TimeoutError`

Publish example
```python
from khorsyio import Bus, Envelope
import msgspec

class Ping(msgspec.Struct):
    x: int

await app.bus.publish("ping", Ping(x=1), source="api")
```

Request/response example
```python
resp = await app.bus.request("user.create", UserIn(name="A"), response_type="user.created")
if resp.is_error:
    # see events.md
    ...
```

Related sections
- [Events: Envelope, Context, Error](events.md)
- [Handlers](handlers.md)
- [App](app.md)
