# Getting Started

See also: [Documentation](README.md)

Minimal application

```python
from khorsyio import App, Response, CorsConfig

app = App(cors=CorsConfig())
app.router.get("/health", lambda req, send: Response.ok(send, status="ok"))

if __name__ == "__main__":
    app.run()
```

How it works
- App combines HTTP, the event bus, and WebSocket transport
- Router resolves the route and calls your handler
- Response.ok sends JSON with status 200

Next topics
- Architecture: [architecture.md](architecture.md)
- HTTP API: [http.md](http.md)
- Event bus: [bus.md](bus.md)
- Events and structs: [events.md](events.md)
- Handlers: [handlers.md](handlers.md)
- Domains and DI: [domain.md](domain.md)
- WebSocket transport: [transport.md](transport.md)
- HTTP client: [client.md](client.md)
- Database: [db.md](db.md)
- SQLAlchemy query utilities: [query.md](query.md)
- Environment settings: [settings.md](settings.md)
