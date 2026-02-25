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

### KV Storage (>= v4.9.2)

```python
await self.put_kv_data("key", value)
value = await self.get_kv_data("key", default)
await self.delete_kv_data("key")
```

### Large File Storage

```python
from astrbot.core.utils.astrbot_path import get_astrbot_data_path

plugin_data_path = get_astrbot_data_path() / "plugin_data" / self.name
plugin_data_path.mkdir(parents=True, exist_ok=True)
```

## Async Tasks
Register long-running or background tasks in `__init__`.

```python
import asyncio
asyncio.create_task(self.my_background_worker())
```

## Accessing Platform Instances

```python
from astrbot.api.platform import AiocqhttpAdapter

platform = self.context.get_platform(filter.PlatformAdapterType.AIOCQHTTP)
assert isinstance(platform, AiocqhttpAdapter)
```

## Calling QQ Protocol API (aiocqhttp)

```python
@filter.command("delete")
async def delete_msg(self, event: AstrMessageEvent):
    if event.get_platform_name() == "aiocqhttp":
        from astrbot.core.platform.sources.aiocqhttp.aiocqhttp_message_event import AiocqhttpMessageEvent
        assert isinstance(event, AiocqhttpMessageEvent)
        client = event.bot
        ret = await client.api.call_action('delete_msg', message_id=event.message_obj.message_id)
        logger.info(f"delete_msg: {ret}")
```

Protocol API docs: [Napcat](https://napcat.apifox.cn/) | [Lagrange](https://lagrange-onebot.apifox.cn/)

## Querying Loaded Plugins and Platforms

```python
# All loaded plugins
plugins = self.context.get_all_stars()  # Returns list of StarMetadata

# All loaded platforms
from astrbot.api.platform import Platform
platforms = self.context.platform_manager.get_insts()  # List[Platform]
```

## Best Practices

- **Docstrings**: Always write a docstring for your handler functions; AstrBot uses them for the help menu.
- **Ruff**: Format your code with `ruff` before submission.
- **Async Everything**: Never use blocking calls like `time.sleep` or `requests.get`. Use `asyncio.sleep` and `httpx.AsyncClient`.
- **Plugin Updates**: If extending an existing plugin's functionality, prefer submitting a PR to that plugin rather than creating a new one.
- **Testing**: Test your plugin thoroughly before publishing.
- **Comments**: Include clear comments in your code.
