# Architecture Overview

See: [Documentation](README.md)

Core elements
- App — entry point, assembles all subsystems, manages lifecycle
- HTTP — Router/Request/Response/CorsConfig, ASGI application with middleware hooks
- Bus — internal event bus with queues, timeouts, and error handling
- Structs — Context, Envelope, Error, serialization convention via msgspec
- Handlers — base class and contract for event processing
- Domain — grouping of handlers and HTTP routes with namespace and DI
- Transport — SocketTransport (ASGI) and SocketClient (socket.io client)
- Client — HttpClient on httpx for outbound HTTP requests
- DB — Database (SQLAlchemy async engine) and query helpers

HTTP request flow
1) ASGI reaches HttpApp
2) Router finds the handler by method and path
3) Your handler works with Request and sends Response

Event flow
1) Source publishes Envelope to Bus (publish or request)
2) Bus dispatches to the matching Handler
3) Handler returns a new Envelope or None
4) Bus continues processing or replies to the initiator

Related sections
- [App](app.md)
- [HTTP](http.md)
- [Bus](bus.md) and [Events](events.md)
- [Handlers](handlers.md), [Domain](domain.md)
- [Transport](transport.md)
- [Database](db.md), [Query helpers](query.md)
