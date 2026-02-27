# Events: Envelope, Context, Error

See: [Documentation](README.md)

Context
- Fields
  - `trace_id` — generated automatically, request-wide identifier
  - `timestamp` — creation time
  - `source` — event source (e.g. `api` or `ws:<sid>`)
  - `user_id` — user identifier
  - `extra` — arbitrary data

Error
- Fields
  - `code` — string error code
  - `message` — description
  - `source` — source that produced the error
  - `trace_id` — for correlating with the original request
  - `details` — extended data

Envelope
- Fields
  - `ctx: Context`
  - `event_type: str`
  - `payload: bytes` — data serialized via msgspec
  - `error: Error | None`
- Methods and properties
  - `create(event_type, data, source="", trace_id=None, user_id="", extra=None)` — build a new envelope
  - `error_from(original, message, code="error", source="", details=None)` — create an error response
  - `decode(typ)` — decode `payload` into type `typ` via msgspec
  - `forward(event_type, data, source="")` — forward a new envelope with the same trace_id, user_id, extra
  - `trace_id` — shortcut property
  - `is_error` — error flag

Usage
- For publishing use `bus.publish("type", data_struct)` — the envelope is built automatically
- For RPC-style calls use `bus.request(...)` — returns an `Envelope`
- Check `resp.is_error` and `resp.error.*` fields

Related sections
- [Event bus](bus.md)
- [Handlers](handlers.md)
