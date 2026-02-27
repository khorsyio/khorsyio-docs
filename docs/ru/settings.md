# Настройки (Settings)

См. оглавление: [Документация](README.md)

Источник настроек
- Значения берутся из переменных окружения и файла .env (загружается через python-dotenv)
- Доступ через объект `settings` из `khorsyio.core.settings`

Разделы настроек
- `settings.server`
  - `host` ENV SERVER_HOST по умолчанию `0.0.0.0`
  - `port` ENV SERVER_PORT по умолчанию `8000`
  - `debug` ENV SERVER_DEBUG по умолчанию `false`
  - `workers` ENV SERVER_WORKERS по умолчанию `1`
- `settings.db`
  - `dsn` ENV DB_DSN по умолчанию `postgresql+asyncpg://localhost:5432/khorsyio`
  - `pool_min` ENV DB_POOL_MIN по умолчанию `2`
  - `pool_max` ENV DB_POOL_MAX по умолчанию `10`
- `settings.bus`
  - `handler_timeout` ENV BUS_HANDLER_TIMEOUT по умолчанию `30.0`

Связанные разделы
- [App](app.md)
- [Database](db.md)
- [Bus](bus.md)
