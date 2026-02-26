# HTTP-клиент (HttpClient)

См. оглавление: [Документация](README.md)

Назначение
- Простой асинхронный HTTP-клиент на базе httpx для исходящих запросов
- Экземпляр доступен как `app.client`

Жизненный цикл
- `await app.client.start()` — создается пул соединений
- `await app.client.stop()` — закрывает соединения
- В `App.startup()`/`shutdown()` это вызывается автоматически

Методы
- `get(url, headers=None, timeout=10.0) -> dict` возвращает `{"status": int, "headers": dict, "body": str}`
- `post(url, data=None, json_data=None, headers=None, timeout=10.0) -> dict`
  - Если указан `json_data`, тело кодируется через msgspec и проставляется `content-type: application/json`

Пример
```python
from khorsyio import App

app = App()

async def fetch_google():
    resp = await app.client.get("https://www.google.com")
    print(resp["status"], len(resp["body"]))
```

Связанные разделы
- [App](app.md)
- [HTTP](http.md)
