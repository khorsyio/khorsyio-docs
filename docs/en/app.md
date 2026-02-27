# Application (App)

See: [Documentation](README.md)

App combines
- Event bus: `app.bus`
- HTTP application and router: `app.router`
- WebSocket transport: `app.transport`
- Database: `app.db`
- Outbound HTTP client: `app.client`

Lifecycle
- `startup()` connects the DB, starts HttpClient, validates the handler graph, starts Bus
- `shutdown()` stops Bus, closes HttpClient and DB
- `run()` starts the ASGI server (granian, otherwise uvicorn)

Registration
- `register(handler)` registers a handler instance on the bus
- `mount(domain)` mounts a Domain: adds handlers and routes with namespace prefix

ASGI
- `__call__` handles `lifespan`, `http`, and `websocket`

Related sections
- [Handlers](handlers.md)
- [Domain](domain.md)
- [HTTP](http.md)
- [Bus](bus.md)
- [Transport](transport.md)
- [Database](db.md)
- [HttpClient](client.md)
- [Settings](settings.md)
