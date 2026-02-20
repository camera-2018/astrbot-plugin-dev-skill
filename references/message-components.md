# AstrBot Message Components

## Overview

A message in AstrBot is a `MessageChain` (a list of message components). Import them via:
`import astrbot.api.message_components as Comp`

## Standard Components

### `Comp.Plain(text)`
Plain text message content.

### `Comp.At(qq)`
At a specific user on QQ or other platforms. Use `event.get_sender_id()` for current sender.

### `Comp.Image.fromURL(url)`
Send an image from a URL.

### `Comp.Image.fromFileSystem(path)`
Send an image from the local file system.

### `Comp.Record(file, url)`
Send a voice message (prefer `.wav` format).

### `Comp.Video.fromFileSystem(path)`, `Comp.Video.fromURL(url)`
Send a video message.

### `Comp.Reply(id)`
Reply to a specific message ID.

### `Comp.Face(id)`
Send a QQ emoji.

### `Comp.Node(uin, name, content)`
A node for group forward/merge messages (mainly `aiocqhttp`).

### `Comp.File(file, name)`
Send a file (platform support varies).

## Creating a Chain

```python
chain = [
    Comp.At(qq=event.get_sender_id()),
    Comp.Plain("Hello, world!"),
    Comp.Image.fromURL("https://example.com/logo.png")
]
yield event.chain_result(chain)
```
