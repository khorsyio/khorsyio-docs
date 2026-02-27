# HTTP: Router, Request, Response, CORS

See: [Documentation](README.md)

Components
- Request — wrapper over the ASGI message for reading body, headers, cookies, and parameters
- Response — utilities for sending responses (JSON, text, error)
- Router — route registration and resolution with path parameter support
- CorsConfig — CORS configuration
- HttpApp — ASGI application, applies middleware hooks and CORS

Request
- `cookies()` -> dict
- `cookie(key, default=None)`
- `header(key, default=None)`
- `body()` -> bytes
- `json(typ: type|None=None)` -> decoded JSON; if `typ` is specified, msgspec is used for validation
- `param(key, default=None)` — query string parameter

Response
- `json(send, data, status=200, headers=None, cookies=None)` — sends application/json
- `ok(send, **kwargs)` — shortcut for JSON 200 response with arbitrary keys
- `text(send, text, status=200, headers=None, cookies=None)`
- `error(send, message, status=400, code="error")` — JSON with `code` and `message` fields

Router
- `add(method, path, handler)` — register a handler
- `get/post/put/delete(path, handler)` — shortcuts
- `use(middleware)` — pre-handler with signature `(req, send, next)`, must call `await next()`
- `after(hook)` — post-hook with signature `(req, res)` where `res` is the sent response dict
- `resolve(method, path)` — used internally by HttpApp
- Path parameters via `/users/{id}` are collected in `req.scope["path_params"]`

CorsConfig
- `origins` — list or `"*"`
- `methods`, `headers`, `credentials`, `max_age`
- `allowed_origin(origin)` — checks the domain

HttpApp
- Constructed as `HttpApp(router, cors=None)`
- ASGI application: `await http(scope, receive, send)`
- Handles `OPTIONS` preflight when CORS is enabled
- Adds CORS headers to responses
- Maintains minimal request tracking

Example
```python
from khorsyio import App, Response

app = App()

async def health(req, send):
    await Response.ok(send, status="ok")

app.router.get("/health", health)
```

Related sections
- [App](app.md)
- [Events](events.md)
- [Handlers](handlers.md)
- [Domains](domain.md)
