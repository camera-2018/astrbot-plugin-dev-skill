---
name: astrbot-plugin-dev
description: Guide for developing AstrBot plugins. Supports creating commands, filters, hooks, handling messages, and integrating with LLMs. Use when requested to create or update an AstrBot plugin.
---

# AstrBot Plugin Development

This skill provides the procedural knowledge to develop AstrBot plugins.

## Quick Start: Basic Command Plugin

Create a `main.py` in your plugin directory:

```python
from astrbot.api.event import filter, AstrMessageEvent
from astrbot.api.star import Context, Star, register
from astrbot.api import logger

@register("helloworld", "YourName", "A simple Hello World plugin", "1.0.0")
class MyPlugin(Star):
    def __init__(self, context: Context):
        super().__init__(context)

    @filter.command("helloworld")
    async def helloworld(self, event: AstrMessageEvent):
        '''A hello world command'''
        user_name = event.get_sender_name()
        yield event.plain_result(f"Hello, {user_name}!")
```

## Core Workflows

### 1. Project Setup and Metadata
A complete plugin requires `metadata.yaml` for identification and `requirements.txt` for dependencies.

- See [references/project-structure.md](references/project-structure.md) for mandatory files and formatting.

### 2. Registering Commands and Filters
Commands are registered using `@filter.command(name)`. You can also filter by event type, platform, or user permission.

- See [references/core-api.md](references/core-api.md) for full list of filters and hooks.

### 3. Handling Messages and Responses
AstrBot uses a "Message Chain" system. You can respond with plain text, images, or a mix of components.

- See [references/message-components.md](references/message-components.md) for how to build complex messages.

### 4. Advanced Integrations
- **Configuration**: Use `_conf_schema.json` for user settings.
- **LLM Tools**: Register tools for LLM to call using `@filter.llm_tool` or `FunctionTool`.
- **Stateful Interaction**: Use `session_waiter` for multi-step prompts.
- **T2I**: Render text or HTML/Jinja2 templates to images.

See [references/advanced-features.md](references/advanced-features.md) for examples.

## Elegant Design Patterns
Follow these patterns for robust, user-friendly plugins:
- Use unified logging.
- Handle errors gracefully to avoid bot crashes.
- Use persistent storage in the global `data/` folder.
- Ensure all IO operations are non-blocking (async).

See [references/patterns.md](references/patterns.md) for detailed code patterns.
