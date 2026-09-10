# Janus Plugin System

> Last updated: September 9, 2026

Janus includes a server-side scripting system powered by **Lua 5.1**.
Plugins are plain `.lua` files that react to server events — chat messages,
connections, file transfers and more — through a simple hook-based API.

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
| `server.bcrypt_hash(password)` | Hash a password with bcrypt (returns `hash, err`) |
| `server.bcrypt_verify(password, hash)` | Check a password against a bcrypt hash |
| `server.set_shared(key, value)` | Set a value shared across all plugins |
| `server.get_shared(key)` | Read a shared value set by any plugin |
| `server.list_files([path])` | List files in the server's file tree |
| `server.file_info(path)` | Get metadata for a single file |
| `server.file_access(path)` | Detect folder type (`"normal"`, `"drop_box"`, or `"upload"`) |
| `server.search_files(opts)` | Query the disk catalog with filters |
| `server.recent_files([limit])` | Get most recently modified files |
| `server.storage_stats()` | Get file tree statistics and breakdowns |
| `server.list_msgboard_posts([limit, offset])` | List message board posts (paginated) |
| `server.get_msgboard_count()` | Get total message board post count |
| `server.list_news_categories([path])` | List news categories at a path |
| `server.list_news_articles(path [, limit, offset])` | List news articles (paginated) |
| `server.get_news_article_count([path])` | Get total article count for a category |

### Available Hooks

Pre-hooks (`pre_*`) can cancel transactions by returning `false`, or modify
messages by returning a string. Post-hooks (`on_*`) are fire-and-forget.

| Hook Name | Trigger |
|-----------|---------|
| `chat` | Chat message sent |
| `message` | Private message sent |
| `broadcast` | Server broadcast |
| `connect` | User completes login |
| `upload` | File upload requested |
| `upload_folder` | Folder upload requested |
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
| `msgboard_post` | Message board post created |
| `news_post` | News article posted |
| `news_delete` | News article deleted |
| `news_category` | News category created |
| `news_bundle` | News bundle (folder) created |
| `news_item_delete` | News item (category/bundle) deleted |
| `voice_join` | User joins a voice chat room |
| `voice_leave` | User leaves a voice chat room |
| `voice_mute` | User mutes or unmutes in voice chat |
| `im` | Messaging-extension instant message sent |
| `friend_request` | User adds someone to their roster |
| `presence` | User changes presence state or status text |
| `call_invite` | User invites others to a call |

The special `on_upload_complete` hook fires after a file upload finishes
transferring. The special `on_upload_folder_complete` hook fires after a
folder upload finishes. Both receive a `ctx` table with the full file `path`
but no `user` table.

The special `on_disconnect` hook fires when a user disconnects (no `ctx`
table, only `user`).

The special `on_msgboard_delete` hook fires when a message board post is
deleted via the REST API. It is a post-hook only (no `pre_` variant) and
receives a `ctx` table with `post_id`.

## Further Reading

- [Plugin Developer Guide](developer-guide.md) — Full API reference with
  examples, hook signatures, data types, and best practices.
