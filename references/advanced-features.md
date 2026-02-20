# AstrBot Advanced Features

## Session Control (Conversation State)

Use `session_waiter` for interactive, multi-step user prompts (e.g.,成语接龙).

```python
from astrbot.core.utils.session_waiter import session_waiter, SessionController

@filter.command("ask-name")
async def ask_name(self, event: AstrMessageEvent):
    yield event.plain_result("What is your name?")

    @session_waiter(timeout=30)
    async def waiter(controller: SessionController, event: AstrMessageEvent):
        name = event.message_str
        await event.send(event.plain_result(f"Hi, {name}!"))
        controller.stop()

    await waiter(event)
```

## LLM Tools (Function Calling)

Register methods for the LLM to call during its reasoning.

```python
@dataclass
class GetWeatherTool(FunctionTool):
    name: str = "get_weather"
    description: str = "Get the current weather for a location."
    parameters: dict = field(
        default_factory=lambda: {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "The city name."},
            },
            "required": ["location"],
        }
    )

    async def run(self, event: AstrMessageEvent, location: str):
        return f"The weather in {location} is sunny."

# Register in __init__:
self.context.add_llm_tools(GetWeatherTool())
```

## Text-to-Image (T2I)

Render text or HTML to an image automatically.

- `url = await self.text_to_image(text)`
- `url = await self.html_render(html_content, data_dict)` (Jinja2 supported)

## Plugin Configuration (`_conf_schema.json`)

Create `_conf_schema.json` in the plugin root to allow users to customize behavior from the WebUI.

```json
{
  "api_key": {
    "description": "API Key",
    "type": "string",
    "hint": "Your service key",
    "default": ""
  },
  "llm_provider": {
    "description": "Select LLM Provider",
    "type": "string",
    "_special": "select_provider"
  },
  "target_persona": {
    "description": "Select Persona",
    "type": "string",
    "_special": "select_persona"
  }
}
```

- `type`: `string`, `int`, `float`, `bool`, `object`, `list`, `text` (for textarea).
- `_special`: `select_provider`, `select_provider_tts`, `select_provider_stt`, `select_persona`.
- Access configuration in `__init__(self, context, config)` as a dictionary: `self.config["api_key"]`.

## LLM Tools (Function Calling)
