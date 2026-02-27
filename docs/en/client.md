# HTTP Client (HttpClient)

See: [Documentation](README.md)

Purpose
- Simple async HTTP client based on httpx for outbound requests
- Instance available as `app.client`

Lifecycle
- `await app.client.start()` — creates the connection pool
- `await app.client.stop()` — closes connections
- Called automatically in `App.startup()` and `shutdown()`

Methods
- `get(url, headers=None, timeout=10.0) -> dict` returns `{"status": int, "headers": dict, "body": str}`
- `post(url, data=None, json_data=None, headers=None, timeout=10.0) -> dict`
  - If `json_data` is provided, the body is encoded via msgspec and `content-type: application/json` is set

Example
```python
from khorsyio import App

app = App()

async def fetch_google():
    resp = await app.client.get("https://www.google.com")
    print(resp["status"], len(resp["body"]))
```

Related sections
- [App](app.md)
- [HTTP](http.md)
