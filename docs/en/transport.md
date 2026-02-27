# WebSocket Transport (SocketTransport / SocketClient)

See: [Documentation](README.md)

SocketTransport
- Purpose: socket.io (asgi) integration for bidirectional events
- Constructor: `SocketTransport(bus: Bus, async_mode: str = "asgi")`
- Internally runs `socketio.AsyncServer` and `socketio.ASGIApp`
- Events
  - `connect(sid, environ)` — registers the connection
  - `disconnect(sid)` — removes the connection
  - `event(sid, data)` — entry point for incoming messages in the format
    `{ "event_type": str, "payload": dict, "trace_id"?: str, "user_id"?: str, "reply_event"?: str }`
    - an `Envelope` is constructed and published to the bus
    - errors are sent back to the client as an `error` event
- Methods
  - `reply_to_sender(envelope)` — if the message arrived via WS and has a `sid`, sends the response to that client
  - `emit(event_name, data, sid=None)` — sends an arbitrary event
  - `emit_envelope(envelope, sid=None)` — serializes and sends as an `event`
  - `connected_sids` — list of currently connected `sid` values

SocketClient
- Purpose: simple socket.io client for subscribing to events and sending messages
- Constructor: `SocketClient(url: str)`
- Methods
  - `on(event_type, callback)` — registers a callback for `event` events
  - `connect()` — establishes the connection
  - `send(event_type, payload, trace_id=None)` — sends an event
  - `disconnect()` — closes the connection

Related sections
- [Bus](bus.md)
- [Events](events.md)
- [App](app.md)
