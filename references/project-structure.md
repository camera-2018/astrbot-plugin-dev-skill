# AstrBot Plugin Project Structure

An elegant plugin should follow this directory structure:

```text
my_plugin/
├── main.py                # Entry point (Required)
├── metadata.yaml          # Plugin metadata (Required)
├── requirements.txt       # Python dependencies (Optional)
├── _conf_schema.json      # Configuration UI schema (Optional)
├── logo.png               # 1:1 ratio logo (Optional)
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

## `requirements.txt`
List any third-party libraries (e.g., `httpx`, `beautifulsoup4`). **Do not** use `requests`; use async-compatible libraries.

## Persistence
Always store persistent data in the global `data/` directory, not within the plugin folder.
```python
import os
# Inside your Star class:
data_dir = os.path.join("data", "plugins", "my_plugin_data")
os.makedirs(data_dir, exist_ok=True)
```
