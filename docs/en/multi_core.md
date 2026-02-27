# Multiprocessing Architecture

`khorsyio` supports multicore task execution based on standard Python `multiprocessing`.

## Why multiprocessing

Standard processes were chosen over isolated sub-interpreters for the following reasons:

1. True parallelism — each process has its own GIL, allowing 100% utilization of each CPU core.
2. Compatibility — full support for `msgspec`, C extensions, database connection pools, and network libraries.
3. Isolation — a crashed worker does not affect the stability of the main application or other workers.

## How it works

The architecture has three levels:

1. Supervisor (Bus) — the main process manages the worker pool (`ProcessPool`).
2. Exchange bus (Pipe) — messages are passed through system pipes. Serialization is done via `msgspec.msgpack` for maximum speed while preserving type information.
3. Workers — isolated processes that dynamically load the required handlers and execute them in their own `asyncio` loop.

## Execution modes

The mode is configured on the handler class:
- `execution_mode = "main"` (default) — runs in the main loop, suitable for I/O tasks
- `execution_mode = "process"` — runs in a separate process from the pool, suitable for CPU-bound tasks

## Example

```python
class HeavyHandler(Handler):
    subscribes_to = "heavy.task"
    publishes = "heavy.result"
    execution_mode = "process"

    async def process(self, data: TaskData, ctx: Context) -> TaskResult:
        # heavy computation does not block the main Bus
        result = some_heavy_math(data.n)
        return TaskResult(value=result)
```
