# Janus Plugin System

Janus includes a server-side scripting system powered by **Lua 5.1** (via
[gopher-lua](https://github.com/yuin/gopher-lua)). Plugins are plain `.lua`
files that react to server events — chat messages, connections, file transfers,
and more; through a simple hook-based API.

## Overview

- **Language:** Lua 5.1
- **Sandboxed:** Plugins run in isolated Lua states with a restricted standard
  library. Dangerous functions (`os.execute`, `os.exit`, `dofile`, etc.) are
  removed at load time.
- **Hot-reloadable:** Plugins can be reloaded without restarting the server via
  the REST API (`POST /api/v2/server/reload` with `"plugins"` target).
- **Pre-hooks and post-hooks:** Pre-hooks (`pre_*`) run before a transaction is
  processed and can cancel or modify it. Post-hooks (`on_*`) run after and are
  used for notifications and logging.

## Quick Start

1. Set `EnablePlugins: true` in `config.yaml`.
2. Place `.lua` files in the `Plugins/` directory (configured via `PluginsDir`,
   defaults to `Plugins/`).
3. Restart the server or trigger a hot-reload.

A minimal plugin:

```lua
-- hello.lua — Say hello when a user connects
function on_connect(user, ctx)
    server.send_message(user.id, "Welcome, " .. user.nick .. "!")
end
```

## Configuration

```yaml
# config.yaml
EnablePlugins: true
PluginsDir: Plugins

# Optional HTTP request filtering for plugins
PluginConfig:
  HTTPAllowlist:
    - "^https://api\\.example\\.com/"
  HTTPBlocklist:
    - "^https?://internal\\."
```

| Setting | Default | Description |
|---------|---------|-------------|
| `EnablePlugins` | `false` | Master switch for the plugin system |
| `PluginsDir` | `Plugins` | Directory to scan for `.lua` files |
| `PluginConfig.HTTPAllowlist` | *(empty)* | Regex patterns — if set, only matching URLs are allowed |
| `PluginConfig.HTTPBlocklist` | *(empty)* | Regex patterns — matching URLs are blocked |

Both allowlist and blocklist are optional. When the allowlist is empty, all
`http://` and `https://` URLs are permitted (unless blocked). When both are
configured, a URL must match the allowlist **and** not match the blocklist.

This is more or less a minor security feature for paranoid operators.

## Plugin API

Plugins interact with the server through the global `server` table. The full
API is documented in the [Plugin Developer Guide](developer-guide.md).

### Core Functions

| Function | Description |
|----------|-------------|
| `server.send_chat(message)` | Send a chat message to all users |
| `server.send_message(uid, message)` | Send a private message to a user |
| `server.broadcast(message)` | Broadcast a server message to all users |
| `server.get_users()` | Get a list of all connected users |
| `server.get_user(uid)` | Get info for a specific user |
| `server.kick(uid [, message])` | Kick a user from the server |
| `server.log(level, message)` | Write to the server log |

### Extended Functions

| Function | Description |
|----------|-------------|
| `server.save(key, value)` | Persist data to disk (per-plugin) |
| `server.load(key)` | Load previously saved data |
| `server.http_get(url)` | Make an HTTP GET request |
| `server.http_post(url, body [, content_type])` | Make an HTTP POST request |
| `server.after(seconds, callback)` | Schedule a one-shot timer |
| `server.every(seconds, callback)` | Schedule a repeating timer |
| `server.cancel_timer(id)` | Cancel a scheduled timer |
| `server.get_account(login)` | Look up an account by login name |
| `server.get_config()` | Read server configuration (safe subset) |
| `server.set_shared(key, value)` | Set a value shared across all plugins |
| `server.get_shared(key)` | Read a shared value set by any plugin |
| `server.list_files([path])` | List files in the server's file tree |
| `server.file_info(path)` | Get metadata for a single file |

### Available Hooks

Pre-hooks (`pre_*`) can cancel transactions by returning `false`, or modify
messages by returning a string. Post-hooks (`on_*`) are fire-and-forget.

| Hook Name | Trigger |
|-----------|---------|
| `chat` | Chat message sent |
| `message` | Private message sent |
| `broadcast` | Server broadcast |
| `connect` | User completes login |
| `upload` | File upload |
| `upload_folder` | Folder upload |
| `download` | File download |
| `download_folder` | Folder download |
| `delete_file` | File deleted |
| `move_file` | File moved |
| `new_folder` | Folder created |
| `set_file_info` | File info updated |
| `new_user` | Account created |
| `delete_user` | Account deleted |
| `disconnect_user` | User kicked/disconnected |
| `user_change` | User changes nick or icon |

The special `on_disconnect` hook fires when a user disconnects (no `ctx`
table, only `user`).

## Example Plugins

Janus ships with a library of example plugins in the `Plugins/` directory,
to showcase what is possible through the plugin system.

## Further Reading

- [Plugin Developer Guide](developer-guide.md) — Full API reference with
  examples, hook signatures, data types, and best practices.
