# AstrBot Core API Reference

## Base Classes

### `Star`
The base class for all AstrBot plugins.
- `context: Context`: The global context object.
- `config: AstrBotConfig`: (Optional) The configuration for the plugin.
- `terminate()`: (Optional) Called when the plugin is unloaded.

### `Context`
The central registry and service provider.
- `send_message(umo, chains)`: Send a message to a session.
- `get_using_provider(umo)`: Get the current LLM provider.
- `get_platform(adapter_type)`: Get a platform adapter instance.
- `add_llm_tools(*tools)`: Register tools for LLM usage.

### `AstrMessageEvent`
Represents an incoming message.
- `message_str`: The plain text content of the message.
- `message_obj`: The `AstrBotMessage` object.
- `unified_msg_origin`: The unique session identifier.
- `get_sender_id()`, `get_sender_name()`: Get sender information.
- `plain_result(text)`, `image_result(path/url)`, `chain_result(chain)`: Create response objects.
- `send(result)`: Send a result manually.
- `stop_event()`: Prevent further processing of this event.

## Decorators

- `@register(name, author, description, version, repo_url)`: Registers the plugin class.
- `@filter.command(name, alias=set(), priority=0)`: Registers a command.
- `@filter.command_group(name)`: Registers a command group.
- `@filter.event_message_type(type)`: Filters by event type (PRIVATE_MESSAGE, GROUP_MESSAGE, ALL).
- `@filter.platform_adapter_type(type)`: Filters by platform (AIOCQHTTP, QQOFFICIAL, etc.).
- `@filter.permission_type(type)`: Filters by user permission (ADMIN).
- `@filter.on_astrbot_loaded()`: Hook for when the bot starts.
- `@filter.on_waiting_llm_request()`: Hook when waiting for LLM request (before lock).
- `@filter.on_llm_request()`: Hook before LLM processing.
- `@filter.on_llm_response()`: Hook after LLM response.
- `@filter.on_decorating_result()`: Hook before sending a message.
- `@filter.llm_tool(name)`: Registers an LLM tool (function calling).
