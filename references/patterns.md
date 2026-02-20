# AstrBot Elegant Plugin Patterns

## Logging
Use the AstrBot-provided logger for unified log management.

```python
from astrbot.api import logger
logger.info("Plugin initialized")
logger.error("Failed to fetch data: %s", error)
```

## Error Handling
Always wrap external API calls and user input in `try-except`. Prevent the entire bot from crashing.

```python
try:
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
        resp.raise_for_status()
except Exception as e:
    logger.error("Network error: %s", e)
    yield event.plain_result(f"Error: {e}")
```

## Persistent Storage
Store data in the global `data/` folder to prevent data loss during plugin updates.

```python
def __init__(self, context: Context):
    self.db_path = os.path.join("data", "plugins", "my_plugin.db")
    # Initialize your DB if it doesn't exist
```

## Async Tasks
Register long-running or background tasks in `__init__`.

```python
import asyncio
asyncio.create_task(self.my_background_worker())
```

## Best Practices
- **Docstrings**: Always write a docstring for your handler functions; AstrBot uses them for the help menu.
- **Ruff**: Format your code with `ruff` before submission.
- **Async Everything**: Never use blocking calls like `time.sleep` or `requests.get`. Use `asyncio.sleep` and `httpx.AsyncClient`.
