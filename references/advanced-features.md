# AstrBot Advanced Features

## Session Control (Conversation State)

Use `session_waiter` for interactive, multi-step user prompts.

```python
from astrbot.core.utils.session_waiter import session_waiter, SessionController
import astrbot.api.message_components as Comp

@filter.command("quiz")
async def quiz(self, event: AstrMessageEvent):
    yield event.plain_result("What is 1+1?")

    @session_waiter(timeout=30, record_history_chains=False)
    async def waiter(controller: SessionController, event: AstrMessageEvent):
        answer = event.message_str
        if answer == "2":
            await event.send(event.plain_result("Correct!"))
            controller.stop()  # End the session immediately
        else:
            await event.send(event.plain_result("Try again!"))
            controller.keep(timeout=30, reset_timeout=True)  # Reset timeout and continue

    try:
        await waiter(event)
    except TimeoutError:
        yield event.plain_result("Time's up!")
    finally:
        event.stop_event()
```

### SessionController

- `keep(timeout, reset_timeout=False)`: Continue the session. If `reset_timeout=True`, restart the timeout counter.
- `stop()`: End the session immediately.
- `get_history_chains()`: Get message chain history (only if `record_history_chains=True`).

### Custom Session ID (Group-Wide Sessions)

By default, sessions are per-sender. To make a group-wide session:

```python
from astrbot.core.utils.session_waiter import session_waiter, SessionFilter, SessionController

class GroupFilter(SessionFilter):
    def filter(self, event: AstrMessageEvent) -> str:
        return event.get_group_id() if event.get_group_id() else event.unified_msg_origin

# In your handler:
await waiter(event, session_filter=GroupFilter())
```

## AI / LLM Integration

### Calling LLM Directly (>= v4.5.7)

```python
@filter.command("ask")
async def ask(self, event: AstrMessageEvent, question: str):
    umo = event.unified_msg_origin
    provider_id = await self.context.get_current_chat_provider_id(umo=umo)
    llm_resp = await self.context.llm_generate(
        chat_provider_id=provider_id,
        prompt=question,
    )
    yield event.plain_result(llm_resp.completion_text)
```

### Calling LLM via Provider (Legacy)

```python
@filter.command("chat")
async def chat(self, event: AstrMessageEvent):
    provider = self.context.get_using_provider(umo=event.unified_msg_origin)
    if provider:
        llm_resp = await provider.text_chat(
            prompt="Hi!",
            context=[
                {"role": "user", "content": "balabala"},
                {"role": "assistant", "content": "response"}
            ],
            system_prompt="You are a helpful assistant."
        )
        yield event.plain_result(llm_resp.completion_text)
```

Additional `text_chat()` parameters:
- `image_urls` (List[str]): Image URLs or file paths for vision models.
- `model` (str): Override the default model.

### Other Provider Types

- STT: `self.context.get_using_stt_provider(umo)` — speech-to-text.
- TTS: `self.context.get_using_tts_provider(umo)` — text-to-speech.
- Embedding: `self.context.get_all_embedding_providers()` — text embedding.

## LLM Tools (Function Calling)

### Dataclass Style (Recommended, >= v4.5.7)

```python
from pydantic import Field
from pydantic.dataclasses import dataclass
from astrbot.core.agent.run_context import ContextWrapper
from astrbot.core.agent.tool import FunctionTool, ToolExecResult
from astrbot.core.astr_agent_context import AstrAgentContext

@dataclass
class GetWeatherTool(FunctionTool[AstrAgentContext]):
    name: str = "get_weather"
    description: str = "Get the current weather for a location."
    parameters: dict = Field(
        default_factory=lambda: {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "The city name."},
            },
            "required": ["city"],
        }
    )

    async def call(self, context: ContextWrapper[AstrAgentContext], **kwargs) -> ToolExecResult:
        city = kwargs["city"]
        return f"The weather in {city} is sunny."

# Register in __init__:
self.context.add_llm_tools(GetWeatherTool())
```

### Decorator Style

```python
@filter.llm_tool(name="get_weather")
async def get_weather(self, event: AstrMessageEvent, location: str):
    '''Get weather info.

    Args:
        location(string): The city name
    '''
    resp = await self.fetch_weather(location)
    yield event.plain_result(f"Weather: {resp}")
```

Supported parameter types: `string`, `number`, `object`, `boolean`, `array`. Since v4.5.7, `array` supports subtypes like `array[string]`.

## Agent and Multi-Agent (>= v4.5.7)

### Tool-Loop Agent

```python
from astrbot.core.agent.tool import ToolSet

@filter.command("agent")
async def agent_cmd(self, event: AstrMessageEvent, query: str):
    umo = event.unified_msg_origin
    prov_id = await self.context.get_current_chat_provider_id(umo)
    llm_resp = await self.context.tool_loop_agent(
        event=event,
        chat_provider_id=prov_id,
        prompt=query,
        tools=ToolSet([GetWeatherTool()]),
        max_steps=30,
        tool_call_timeout=60,
    )
    yield event.plain_result(llm_resp.completion_text)
```

### Multi-Agent

Define sub-agents as `FunctionTool` classes that internally call `self.context.tool_loop_agent()`, then compose them:

```python
@dataclass
class SubAgent(FunctionTool[AstrAgentContext]):
    name: str = "weather_agent"
    description: str = "Agent that handles weather queries"
    parameters: dict = Field(default_factory=lambda: {
        "type": "object",
        "properties": {"query": {"type": "string", "description": "The query"}},
        "required": ["query"],
    })

    async def call(self, context: ContextWrapper[AstrAgentContext], **kwargs) -> ToolExecResult:
        ctx = context.context.context
        event = context.context.event
        llm_resp = await ctx.tool_loop_agent(
            event=event,
            chat_provider_id=await ctx.get_current_chat_provider_id(event.unified_msg_origin),
            prompt=kwargs["query"],
            tools=ToolSet([GetWeatherTool()]),
            max_steps=30,
        )
        return llm_resp.completion_text

# Then use in a parent agent:
llm_resp = await self.context.tool_loop_agent(
    event=event,
    chat_provider_id=prov_id,
    prompt="What's the weather in Beijing?",
    system_prompt="You are the main agent. Delegate to sub-agents.",
    tools=ToolSet([SubAgent()]),
    max_steps=30,
)
```

## Conversation Manager

```python
from astrbot.core.conversation_mgr import Conversation

conv_mgr = self.context.conversation_manager
uid = event.unified_msg_origin
curr_cid = await conv_mgr.get_curr_conversation_id(uid)
conversation = await conv_mgr.get_conversation(uid, curr_cid)
```

Key methods:
- `new_conversation(umo, ...)`: Create a new conversation and switch to it. Returns the new UUID conversation ID.
- `switch_conversation(umo, conversation_id)`: Switch to a specific conversation.
- `delete_conversation(umo, conversation_id=None)`: Delete a conversation.
- `get_curr_conversation_id(umo)`: Get the current conversation ID.
- `get_conversation(umo, cid, create_if_not_exists=False)`: Get a conversation object.
- `get_conversations(umo, platform_id)`: List all conversations.
- `update_conversation(umo, cid, history, title, persona_id)`: Update conversation data.

## Persona Manager

```python
persona_mgr = self.context.persona_manager
```

Key methods:
- `get_persona(persona_id)`: Get a persona by ID.
- `get_all_personas()`: Get all personas.
- `create_persona(persona_id, system_prompt, begin_dialogs, tools)`: Create a new persona.
- `update_persona(persona_id, system_prompt, begin_dialogs, tools)`: Update a persona.
- `delete_persona(persona_id)`: Delete a persona.
- `get_default_persona_v3(umo)`: Get the default persona for a session.

## Text-to-Image (T2I)

### Basic

```python
url = await self.text_to_image(text)
yield event.image_result(url)
```

### Custom HTML + Jinja2

```python
TMPL = '''
<div style="font-size: 32px;">
<h1>Todo List</h1>
<ul>
{% for item in items %}
    <li>{{ item }}</li>
{% endfor %}
</ul>
</div>
'''

@filter.command("todo")
async def todo(self, event: AstrMessageEvent):
    url = await self.html_render(TMPL, {"items": ["Item 1", "Item 2", "Item 3"]})
    yield event.image_result(url)
```

You can use the [AstrBot Text2Image Playground](https://t2i-playground.astrbot.app/) to preview templates.

Render options follow the Playwright `screenshot` API:
- `timeout`, `type` ("jpeg"/"png"), `quality`, `omit_background`, `full_page`, `scale`, etc.

## Plugin Configuration (`_conf_schema.json`)

Create `_conf_schema.json` in the plugin root to allow users to customize behavior from the WebUI.

### Schema Fields

- `type` (**required**): `string`, `text`, `int`, `float`, `bool`, `object`, `list`, `dict`, `file`, `template_list`.
- `description`: Short description of the config item.
- `hint`: Tooltip text shown on hover.
- `obvious_hint`: If `true`, the hint is displayed prominently.
- `default`: Default value.
- `items`: Sub-schema for `object` type (nestable).
- `invisible`: If `true`, hidden from the WebUI.
- `options`: Dropdown options list, e.g. `["chat", "agent"]`.
- `editor_mode`: Enable code editor (>= v3.5.10).
- `editor_language`: Code language for editor (default: `json`).
- `editor_theme`: `vs-light` (default) or `vs-dark`.
- `_special`: `select_provider`, `select_provider_tts`, `select_provider_stt`, `select_persona` (>= v4.0.0).

### Basic Example

```json
{
  "api_key": {
    "description": "API Key",
    "type": "string",
    "hint": "Your service key",
    "default": ""
  },
  "mode": {
    "description": "Operation Mode",
    "type": "string",
    "options": ["chat", "agent", "workflow"]
  },
  "llm_provider": {
    "description": "Select LLM Provider",
    "type": "string",
    "_special": "select_provider"
  }
}
```

### File Upload (>= v4.13.0)

```json
{
  "demo_files": {
    "type": "file",
    "description": "Upload files",
    "default": [],
    "file_types": ["pdf", "docx"]
  }
}
```

### Dict Type

```json
{
  "custom_params": {
    "type": "dict",
    "description": "Custom parameters",
    "items": {},
    "template_schema": {
      "temperature": {
        "name": "Temperature",
        "type": "float",
        "default": 0.6,
        "slider": {"min": 0, "max": 2, "step": 0.1}
      }
    }
  }
}
```

### Template List (>= v4.10.4)

```json
{
  "rules": {
    "type": "template_list",
    "description": "Rule definitions",
    "templates": {
      "simple_rule": {
        "name": "Simple Rule",
        "items": {
          "pattern": {"type": "string", "description": "Match pattern"},
          "enabled": {"type": "bool", "default": true}
        }
      }
    }
  }
}
```

### Accessing Config

```python
from astrbot.api import AstrBotConfig

@register("myplugin", "Author", "Description", "1.0.0")
class MyPlugin(Star):
    def __init__(self, context: Context, config: AstrBotConfig):
        super().__init__(context)
        self.config = config  # AstrBotConfig inherits from dict
        api_key = self.config["api_key"]
        self.config.save_config()  # Persist config changes
```
