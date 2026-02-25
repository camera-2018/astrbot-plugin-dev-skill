# AstrBot Core API Reference

## Base Classes

### `Star`

The base class for all AstrBot plugins.

- `context: Context`: The global context object.
- `config: AstrBotConfig`: (Optional) Plugin configuration, passed in `__init__` when `_conf_schema.json` exists.
- `name: str`: Plugin name (available >= v4.9.2).
- `terminate()`: (Optional) Called when the plugin is unloaded or disabled.
- `text_to_image(text, return_url=True)`: Render text to an image. Returns a URL or local path.
- `html_render(html, data, options={})`: Render HTML+Jinja2 template to an image.
- `put_kv_data(key, value)`: Store a key-value pair (>= v4.9.2).
- `get_kv_data(key, default)`: Retrieve a KV value (>= v4.9.2).
- `delete_kv_data(key)`: Delete a KV entry (>= v4.9.2).

### `Context`

The central registry and service provider.

- `send_message(umo, chains)`: Send a proactive message to a session.
- `get_using_provider(umo)`: Get the current LLM provider for a session.
- `get_provider_by_id(provider_id)`: Get an LLM provider by its ID.
- `get_all_providers()`: Get all LLM providers.
- `get_current_chat_provider_id(umo)`: Get the current chat model provider ID (>= v4.5.7).
- `llm_generate(chat_provider_id, prompt, ...)`: Call an LLM directly (>= v4.5.7).
- `tool_loop_agent(event, chat_provider_id, prompt, tools, ...)`: Run a tool-loop agent (>= v4.5.7).
- `get_platform(adapter_type)`: Get a platform adapter instance.
- `add_llm_tools(*tools)`: Register tools for LLM function calling (>= v4.5.1).
- `get_llm_tool_manager()`: Get the LLM Tool Manager (contains all registered tools).
- `get_using_stt_provider(umo)`: Get the current STT provider.
- `get_using_tts_provider(umo)`: Get the current TTS provider.
- `get_all_stt_providers()`, `get_all_tts_providers()`, `get_all_embedding_providers()`: Get all providers of a type.
- `conversation_manager`: Access the `ConversationManager`.
- `persona_manager`: Access the `PersonaManager`.
- `platform_manager.get_insts()`: Get all loaded platform instances.
- `get_all_stars()`: Get all loaded plugins (`StarMetadata`).

### `AstrMessageEvent`

Represents an incoming message event.

- `message_str`: The plain text content of the message.
- `message_obj`: The `AstrBotMessage` object.
- `unified_msg_origin`: The unique session identifier (format: `platform_name:message_type:session_id`).
- `get_sender_id()`: Get sender's platform user ID.
- `get_sender_name()`: Get sender's display name.
- `get_group_id()`: Get group ID (empty string for private messages).
- `get_platform_name()`: Get the platform adapter name (e.g., `"aiocqhttp"`).
- `plain_result(text)`: Create a plain text response.
- `image_result(path_or_url)`: Create an image response.
- `chain_result(chain)`: Create a message chain response.
- `make_result()`: Create an empty `MessageEventResult` to build manually.
- `send(result)`: Send a result immediately (for use in hooks/session waiters where `yield` is not allowed).
- `get_result()`: Get the current result (used in `on_decorating_result` hook).
- `stop_event()`: Prevent further processing of this event by other plugins or the LLM.

### `AstrBotMessage`

The message object from the platform adapter. Access via `event.message_obj`.

```python
class AstrBotMessage:
    type: MessageType          # Message type (PRIVATE_MESSAGE, GROUP_MESSAGE)
    self_id: str               # Bot's ID
    session_id: str            # Session ID
    message_id: str            # Message ID
    group_id: str              # Group ID (empty for private messages)
    sender: MessageMember      # Sender info
    message: List[BaseMessageComponent]  # Message chain
    message_str: str           # Plain text (all Plain segments concatenated)
    raw_message: object        # Platform-specific raw message object
    timestamp: int             # Message timestamp
```

## Decorators

### Plugin Registration

- `@register(name, author, description, version, repo_url)`: Registers the plugin class. This is required for AstrBot to detect and load the plugin. Priority is lower than `metadata.yaml`.

### Commands

- `@filter.command(name, alias=set(), priority=0)`: Registers a command.
- `@filter.command_group(name, alias=set())`: Registers a command group.

#### Command Parameters (Auto-Parsed)

```python
@filter.command("add")
async def add(self, event: AstrMessageEvent, a: int, b: int):
    # /add 1 2 -> Result: 3
    yield event.plain_result(f"Result: {a + b}")
```

#### Command Groups (Nestable)

```python
@filter.command_group("math")
def math(self):
    pass

@math.command("add")
async def add(self, event: AstrMessageEvent, a: int, b: int):
    yield event.plain_result(f"Result: {a + b}")

# Nested groups use .group() instead of .command_group()
@math.group("calc")
def calc():
    pass

@calc.command("sum")
async def calc_sum(self, event: AstrMessageEvent, a: int, b: int):
    # /math calc sum 1 2
    yield event.plain_result(f"Result: {a + b}")
```

#### Command Aliases

```python
@filter.command("help", alias={'帮助', 'helpme'})
async def help_cmd(self, event: AstrMessageEvent):
    yield event.plain_result("Available commands: ...")
```

### Event Filters

- `@filter.event_message_type(type)`: Filters by event type (`EventMessageType.PRIVATE_MESSAGE`, `GROUP_MESSAGE`, `ALL`).
- `@filter.platform_adapter_type(type)`: Filters by platform (`PlatformAdapterType.AIOCQHTTP`, `QQOFFICIAL`, `GEWECHAT`, `ALL`). Supports bitwise OR: `PlatformAdapterType.AIOCQHTTP | PlatformAdapterType.QQOFFICIAL`.
- `@filter.permission_type(type)`: Filters by user permission (`PermissionType.ADMIN`).

#### Multiple Filters (AND Logic)

```python
@filter.command("secret")
@filter.permission_type(filter.PermissionType.ADMIN)
@filter.event_message_type(filter.EventMessageType.PRIVATE_MESSAGE)
async def secret_cmd(self, event: AstrMessageEvent):
    yield event.plain_result("Admin-only private command!")
```

### Event Hooks

Event hooks **cannot** be combined with `@filter.command`, `@filter.command_group`, `@filter.event_message_type`, etc.

In hooks, **do not** use `yield` to send messages. Use `await event.send(result)` instead.

- `@filter.on_astrbot_loaded()`: Hook for when the bot finishes initialization.
- `@filter.on_waiting_llm_request()`: Hook when waiting for LLM request (before acquiring the session lock). Good for sending "thinking..." feedback.
- `@filter.on_llm_request()`: Hook before LLM processing. Receives `(self, event, req: ProviderRequest)`. Can modify `req.system_prompt`, etc.
- `@filter.on_llm_response()`: Hook after LLM response. Receives `(self, event, resp: LLMResponse)`.
- `@filter.on_decorating_result()`: Hook before sending a message. Modify `event.get_result().chain`.
- `@filter.after_message_sent()`: Hook after a message has been sent to the platform.

### LLM Tools

- `@filter.llm_tool(name)`: Registers an LLM tool via decorator with docstring parsing.

### Priority

All commands, filters, and hooks support a `priority` parameter (default `0`). Higher priority executes first.

```python
@filter.command("important", priority=10)
async def important(self, event: AstrMessageEvent):
    yield event.plain_result("I run first!")
```

## Controlling Event Propagation

```python
@filter.command("check")
async def check(self, event: AstrMessageEvent):
    if not self.is_valid():
        yield event.plain_result("Check failed")
        event.stop_event()  # Stop all further processing (other plugins, LLM, etc.)
```

## Platform Compatibility Matrix

| Platform              | At  | Plain | Image | Record | Video | Reply | Proactive Messages |
| --------------------- | --- | ----- | ----- | ------ | ----- | ----- | ------------------ |
| QQ (aiocqhttp)        | ✅  | ✅    | ✅    | ✅     | ✅    | ✅    | ✅                 |
| Telegram              | ✅  | ✅    | ✅    | ✅     | ✅    | ✅    | ✅                 |
| QQ Official           | ❌  | ✅    | ✅    | ❌     | ❌    | ❌    | ❌                 |
| Lark (飞书)           | ✅  | ✅    | ✅    | ❌     | ❌    | ✅    | ✅                 |
| WeCom (企业微信)      | ❌  | ✅    | ✅    | ✅     | ❌    | ❌    | ❌                 |
| DingTalk              | ❌  | ✅    | ✅    | ❌     | ❌    | ❌    | ❌                 |

- QQ (aiocqhttp) supports all message types, including `Poke`, `Node(s)` (forwarded messages).
- QQ Official and DingTalk auto-prepend `At` when sending messages.
- DingTalk images only support HTTP URLs.
