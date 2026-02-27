# Settings

See: [Documentation](README.md)

Source of settings
- Values are read from environment variables and a .env file (loaded via python-dotenv)
- Accessed via the `settings` object from `khorsyio.core.settings`

Settings sections
- `settings.server`
  - `host` ENV SERVER_HOST default `0.0.0.0`
  - `port` ENV SERVER_PORT default `8000`
  - `debug` ENV SERVER_DEBUG default `false`
  - `workers` ENV SERVER_WORKERS default `1`
- `settings.db`
  - `dsn` ENV DB_DSN default `postgresql+asyncpg://localhost:5432/khorsyio`
  - `pool_min` ENV DB_POOL_MIN default `2`
  - `pool_max` ENV DB_POOL_MAX default `10`
- `settings.bus`
  - `handler_timeout` ENV BUS_HANDLER_TIMEOUT default `30.0`

Related sections
- [App](app.md)
- [Database](db.md)
- [Bus](bus.md)
