# AstrBot Message Components

## Overview

A message in AstrBot is a `MessageChain` (a list of message components). Import them via:
`import astrbot.api.message_components as Comp`

## Standard Components

### `Comp.Plain(text)`
Plain text message content.

> In the aiocqhttp adapter, `Plain` text is auto-stripped of leading/trailing whitespace. Use zero-width space `\u200b` to preserve spacing.

### `Comp.At(qq)`
At a specific user on QQ or other platforms. Use `event.get_sender_id()` for current sender.

### `Comp.Image.fromURL(url)`
Send an image from a URL. URL must start with `http` or `https`.

### `Comp.Image.fromFileSystem(path)`
Send an image from the local file system.

### `Comp.Record(file, url)`
Send a voice message. Only `.wav` format is currently supported; convert other formats yourself.

### `Comp.Video.fromFileSystem(path)`, `Comp.Video.fromURL(url)`
Send a video message.

### `Comp.Reply(id)`
Reply to a specific message ID.

### `Comp.Face(id)`
Send a QQ emoji. ID reference: https://bot.q.qq.com/wiki/develop/api-v2/openapi/emoji/model.html#EmojiType

### `Comp.Node(uin, name, content)`
A node for group forwarded (merge) messages. Mainly supported on OneBot v11 (aiocqhttp).

```python
node = Comp.Node(
    uin=123456,
    name="BotName",
    content=[Comp.Plain("hello"), Comp.Image.fromFileSystem("test.jpg")]
)
yield event.chain_result([node])
```

### `Comp.File(file, name)`
Send a file (platform support varies).

```python
Comp.File(file="path/to/file.txt", name="file.txt")
```

## Creating a Chain (Passive Response)

```python
chain = [
    Comp.At(qq=event.get_sender_id()),
    Comp.Plain("Hello, world!"),
    Comp.Image.fromURL("https://example.com/logo.png")
]
yield event.chain_result(chain)
```

## Proactive (Active) Messaging

Proactive messages are sent by the bot without a triggering user message. Some platforms may not support this.

Use `event.unified_msg_origin` to capture the session identifier, store it, then send later via `self.context.send_message()`.

```python
from astrbot.api.event import MessageChain

@filter.command("subscribe")
async def subscribe(self, event: AstrMessageEvent):
    umo = event.unified_msg_origin
    # Store umo for later use (e.g., in a database or dict)
    self.subscriptions.append(umo)
    yield event.plain_result("Subscribed!")

# Later, in a background task or scheduled job:
async def notify_subscribers(self):
    for umo in self.subscriptions:
        message_chain = MessageChain().message("New update!").file_image("path/to/image.jpg")
        await self.context.send_message(umo, message_chain)
```

`unified_msg_origin` is a string that uniquely identifies a session so AstrBot can route the message to the correct platform and conversation.

## Platform Compatibility Matrix

| Platform              | At  | Plain | Image | Record | Video | Reply | Proactive |
| --------------------- | --- | ----- | ----- | ------ | ----- | ----- | --------- |
| QQ (aiocqhttp)        | ✅  | ✅    | ✅    | ✅     | ✅    | ✅    | ✅        |
| Telegram              | ✅  | ✅    | ✅    | ✅     | ✅    | ✅    | ✅        |
| QQ Official           | ❌  | ✅    | ✅    | ❌     | ❌    | ❌    | ❌        |
| Lark (飞书)           | ✅  | ✅    | ✅    | ❌     | ❌    | ✅    | ✅        |
| WeCom (企业微信)      | ❌  | ✅    | ✅    | ✅     | ❌    | ❌    | ❌        |
| DingTalk              | ❌  | ✅    | ✅    | ❌     | ❌    | ❌    | ❌        |
