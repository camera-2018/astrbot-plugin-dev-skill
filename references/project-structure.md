# AstrBot Plugin Project Structure

## Plugin Naming Conventions

- Name should start with `astrbot_plugin_` (recommended).
- No spaces allowed.
- All lowercase letters.
- Keep it short.

## Directory Structure

```text
my_plugin/
├── main.py                # Entry point (Required)
├── metadata.yaml          # Plugin metadata (Required)
├── requirements.txt       # Python dependencies (Optional)
├── _conf_schema.json      # Configuration UI schema (Optional)
├── logo.png               # 1:1 ratio logo, 256x256 recommended (Optional)
└── README.md              # User documentation (Optional)
```

## `metadata.yaml`

This file is used by the AstrBot Plugin Market.

```yaml
name: my_plugin_name
display_name: My Elegant Plugin
author: YourName
version: 1.0.0
description: A detailed description of what this plugin does.
repo_url: https://github.com/user/my_plugin
astrbot_version: ">=4.16,<5"
support_platforms:
  - aiocqhttp
  - telegram
```

### `support_platforms`

Supported platform adapter keys:

- `aiocqhttp` — QQ via OneBot v11
- `qq_official` — QQ Official API
- `telegram`
- `wecom` — WeCom (企业微信)
- `lark` — Lark/Feishu (飞书)
- `dingtalk`
- `discord`
- `slack`
- `kook`
- `vocechat`
- `weixin_official_account`
- `satori`
- `misskey`
- `line`

### `astrbot_version`

Follows PEP 440 version constraints (no `v` prefix). Examples:

- `>=4.17.0`
- `>=4.16,<5`
- `~=4.17`

When the running AstrBot version does not satisfy the range, the plugin will be blocked from loading.

## `requirements.txt`

List any third-party libraries (e.g., `httpx`, `beautifulsoup4`). **Do not** use `requests`; use async-compatible libraries like `httpx` or `aiohttp`.

## Development Environment Setup

```bash
git clone https://github.com/AstrBotDevs/AstrBot
mkdir -p AstrBot/data/plugins
cd AstrBot/data/plugins
git clone <your-plugin-repo-url>
```

Open the `AstrBot` project in VSCode. Edit code in `data/plugins/<your_plugin>/`.

### Debugging

AstrBot injects plugins at runtime. After modifying plugin code, open the AstrBot WebUI plugin management page, click the `...` button on your plugin, then select `Reload Plugin` (重载插件).

If the plugin fails to load due to code errors, you can click **"Try One-Click Reload Fix"** in the error prompt.

## Publishing to the Plugin Market

1. Push your plugin code to GitHub.
2. Visit the [AstrBot Plugin Market](https://plugins.astrbot.app).
3. Click the `+` button, fill in the plugin info, and click `Submit to GitHub`.
4. You will be directed to an Issue page on the AstrBot repo — confirm and create the Issue.

## Persistence

Store persistent data in the global `data/` directory, not within the plugin folder, to prevent data loss during plugin updates.

### Simple KV Storage (>= v4.9.2)

```python
# Inside your Star class handlers:
await self.put_kv_data("key", "value")
value = await self.get_kv_data("key", default_value)
await self.delete_kv_data("key")
```

### Large File Storage

```python
from astrbot.core.utils.astrbot_path import get_astrbot_data_path

plugin_data_path = get_astrbot_data_path() / "plugin_data" / self.name
```
