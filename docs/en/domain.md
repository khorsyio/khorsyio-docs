# Domains and Namespaces

See: [Documentation](README.md)

Purpose
- Grouping event handlers and HTTP routes into functional modules
- Automatic prefixing of event types via `namespace`
- Simplified DI when creating handlers

Key fields
- `handlers: list` — classes or instances of `Handler`
- `routes: list[Route]` — HTTP routes
- `namespace: str` — prefix for `subscribes_to` and `publishes`

Behavior of `setup(app)`
- For each element in `handlers`
  - if it is a class, it is instantiated with DI by constructor parameter names
  - if it is an instance, it is used as is
  - if `namespace` is set, `"{namespace}."` is prepended to `subscribes_to` and `publishes`
  - the handler is registered on `app.bus`
- Routes are mounted on `app.router`

Example
```python
from khorsyio import Domain, Route

users = Domain()
users.namespace = "user"
users.handlers = [CreateUser]
users.routes = [Route("GET", "/users", get_users)]

app.mount(users)
```

DI name mapping
- `db -> app.db`
- `client -> app.client`
- `bus -> app.bus`
- `transport -> app.transport`
- `app -> app`
- any other name -> `app`

Related sections
- [Handlers](handlers.md)
- [HTTP](http.md)
- [App](app.md)
