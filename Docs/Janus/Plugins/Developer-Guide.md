# Janus Plugin Developer Guide

This document is a complete reference for developing Lua plugins for the Janus
Hotline server. It covers the hook system, the full server API, data types,
configuration, and best practices.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Plugin Lifecycle](#plugin-lifecycle)
3. [Hooks](#hooks)
   - [Pre-Hooks](#pre-hooks)
   - [Post-Hooks](#post-hooks)
   - [Disconnect Hook](#disconnect-hook)
   - [Hook Reference Table](#hook-reference-table)
4. [The `user` Table](#the-user-table)
5. [The `ctx` Table](#the-ctx-table)
6. [Server API Reference](#server-api-reference)
   - [Chat & Messaging](#chat--messaging)
   - [User Management](#user-management)
   - [Logging](#logging)
   - [Persistent Storage](#persistent-storage)
   - [HTTP Requests](#http-requests)
   - [Timers](#timers)
   - [Account & Config Access](#account--config-access)
   - [Inter-Plugin Communication](#inter-plugin-communication)
   - [File System Access](#file-system-access)
7. [Sandbox Environment](#sandbox-environment)
8. [Limits & Constraints](#limits--constraints)
9. [Configuration Reference](#configuration-reference)
10. [Best Practices](#best-practices)
11. [Examples](#examples)

---

## Getting Started

A Janus plugin is a plain `.lua` file placed in the server's `Plugins/`
directory. Each plugin runs in its own isolated Lua 5.1 state. Plugins react to
server events by defining hook functions at the global scope.

### Minimal Plugin Example

```lua
-- welcome.lua
function on_connect(user, ctx)
    server.send_message(user.id, "Welcome, " .. user.nick .. "!")
end
```

### Enabling Plugins

Add the following to `config.yaml`:

```yaml
EnablePlugins: true
PluginsDir: Plugins    # server default, needs only be defined if placed elsewhere
```

Restart the server or reload plugins via the REST API:

```
POST /api/v2/server/reload
{"target": "plugins"}
```

### File Naming

- Plugins are loaded in alphabetical order by filename.
- Only files ending in `.lua` are loaded (other files are ignored).
- The filename (minus `.lua`) is used as the plugin's identity for persistent
  storage and logging.

---

## Plugin Lifecycle

1. **Load** — On startup (or reload), the server scans `PluginsDir` for `.lua`
   files. Each file is executed in a fresh Lua state with the `server` API
   already registered.
2. **Run** — Global function definitions (hooks) are registered. Top-level code
   outside functions runs once at load time — useful for initializing state.
3. **Hook calls** — As server events occur, matching hook functions are called
   across all loaded plugins in load order.
4. **Reload** — On reload, all plugin states are closed (timers cancelled,
   Lua states garbage collected) and plugins are re-loaded from disk. Shared
   state (`set_shared`/`get_shared`) is cleared on reload. Persistent storage
   (`save`/`load`) survives reloads.
5. **Shutdown** — On server shutdown, all timers are cancelled and Lua states
   are closed.

---

## Hooks

Hooks are global Lua functions with specific names. Two types exist:

### Pre-Hooks

Pre-hooks run **before** the server processes a transaction. They can inspect
and optionally cancel or modify the transaction.

**Naming:** `pre_<event>` (e.g., `pre_chat`, `pre_upload`)

**Signature:**
```lua
function pre_chat(user, ctx)
    -- Return false to cancel the transaction.
    -- Return a string to replace the message content.
    -- Return true/nil to allow it unchanged.
end
```

**Return values:**

| Return | Effect |
|--------|--------|
| `false` | Cancel the transaction — it will not be processed |
| A string | Replace the `FieldData` (message content) and continue |
| `true`, `nil`, or no return | Allow the transaction unchanged |

Pre-hooks are called in plugin load order. If any plugin cancels a transaction
(`return false`), later plugins' pre-hooks for that event are **not** called.

**Message modification example:**

```lua
function pre_chat(user, ctx)
    -- Censor a word: the replaced string becomes the new message.
    return string.gsub(ctx.message, "badword", "****")
end
```

> **Note:** `string.gsub` returns two values (the modified string and a count).
> Only the first return value is used — extra values are ignored.

### Post-Hooks

Post-hooks run **after** the server has processed a transaction. They are
fire-and-forget — return values are ignored.

**Naming:** `on_<event>` (e.g., `on_chat`, `on_upload`)

**Signature:**
```lua
function on_chat(user, ctx)
    server.log("info", user.nick .. " said: " .. ctx.message)
end
```

### Disconnect Hook

The `on_disconnect` hook fires when any user disconnects. Unlike other hooks,
it receives only the `user` table — there is no `ctx`.

```lua
function on_disconnect(user)
    server.log("info", user.nick .. " left the server")
end
```

### Hook Reference Table

| Event | Pre-Hook | Post-Hook | Trigger |
|-------|----------|-----------|---------|
| `chat` | `pre_chat` | `on_chat` | Chat message sent |
| `message` | `pre_message` | `on_message` | Private message sent |
| `broadcast` | `pre_broadcast` | `on_broadcast` | Server broadcast |
| `connect` | `pre_connect` | `on_connect` | User completes login |
| `upload` | `pre_upload` | `on_upload` | File upload started |
| `upload_folder` | `pre_upload_folder` | `on_upload_folder` | Folder upload started |
| `download` | `pre_download` | `on_download` | File download started |
| `download_folder` | `pre_download_folder` | `on_download_folder` | Folder download started |
| `delete_file` | `pre_delete_file` | `on_delete_file` | File deleted |
| `move_file` | `pre_move_file` | `on_move_file` | File moved |
| `new_folder` | `pre_new_folder` | `on_new_folder` | Folder created |
| `set_file_info` | `pre_set_file_info` | `on_set_file_info` | File info updated |
| `new_user` | `pre_new_user` | `on_new_user` | Account created |
| `delete_user` | `pre_delete_user` | `on_delete_user` | Account deleted |
| `disconnect_user` | `pre_disconnect_user` | `on_disconnect_user` | User kicked |
| `user_change` | `pre_user_change` | `on_user_change` | User changes nick/icon |
| *(disconnect)* | — | `on_disconnect` | User disconnects (no `ctx`) |

---

## The `user` Table

Every hook receives a `user` table as its first argument. This describes the
user who triggered the event.

| Field | Type | Description |
|-------|------|-------------|
| `id` | number | Unique client ID (0–65535) |
| `nick` | string | Display name |
| `login` | string | Account login name (empty for guests) |
| `ip` | string | Client IP address |
| `icon` | number | User icon ID |
| `version` | number | Client version number |

**Example:**

```lua
function on_connect(user, ctx)
    server.log("info", string.format(
        "%s (login=%s) connected from %s with icon %d",
        user.nick, user.login, user.ip, user.icon
    ))
end
```

---

## The `ctx` Table

The second argument to most hooks is a `ctx` (context) table containing
event-specific data. Its fields vary by hook type:

### Chat (`pre_chat` / `on_chat`)

| Field | Type | Description |
|-------|------|-------------|
| `message` | string | The chat message text |
| `emote` | boolean | `true` if the message is an emote (`/me` action) |
| `chat_id` | number | Chat room ID (0 = public chat) |

### Private Message (`pre_message` / `on_message`)

| Field | Type | Description |
|-------|------|-------------|
| `message` | string | The message text |
| `target_id` | number | Recipient's user ID |

### Broadcast (`pre_broadcast` / `on_broadcast`)

| Field | Type | Description |
|-------|------|-------------|
| `message` | string | The broadcast text |

### Connect (`pre_connect` / `on_connect`)

| Field | Type | Description |
|-------|------|-------------|
| `nick` | string | The user's nickname |
| `login` | string | The user's account login |

### File Operations

Applies to: `upload`, `upload_folder`, `download`, `download_folder`,
`delete_file`, `move_file`, `new_folder`, `set_file_info`

| Field | Type | Description |
|-------|------|-------------|
| `filename` | string | Name of the file or folder |

### Disconnect User (`pre_disconnect_user` / `on_disconnect_user`)

| Field | Type | Description |
|-------|------|-------------|
| `target_id` | number | ID of the user being disconnected |

### User Change (`pre_user_change` / `on_user_change`)

| Field | Type | Description |
|-------|------|-------------|
| `nick` | string | New nickname (if changed) |
| `icon` | number | New icon ID (if changed) |

### Disconnect (`on_disconnect`)

No `ctx` table — only the `user` table is passed.

---

## Server API Reference

All API functions are accessed through the global `server` table.

### Chat & Messaging

#### `server.send_chat(message)`

Sends a chat message to all users with chat read permission. The message
appears as a server announcement (prefixed with `*** `).

```lua
server.send_chat("Server will restart in 5 minutes")
```

#### `server.send_message(user_id, message)`

Sends a private message to a specific user.

**Returns:** `true` if the user was found, `false` otherwise.

```lua
server.send_message(user.id, "Hello, " .. user.nick)
```

#### `server.broadcast(message)`

Sends a server broadcast message to all connected users. This uses the
Hotline broadcast mechanism, which most clients display as a modal dialog.

```lua
server.broadcast("Server maintenance in 10 minutes!")
```

### User Management

#### `server.get_users()`

Returns a table (array) of all connected users. Each element has the same
fields as the [`user` table](#the-user-table).

```lua
local users = server.get_users()
for _, u in ipairs(users) do
    server.log("info", u.nick .. " is online")
end
```

#### `server.get_user(user_id)`

Returns the user table for a specific user ID, or `nil` if not found.

```lua
local target = server.get_user(42)
if target then
    server.send_chat(target.nick .. " is online")
end
```

#### `server.kick(user_id [, message])`

Kicks a user from the server. If a message is provided, it's sent to the user
before disconnection.

**Returns:** `true` if the user was found, `false` otherwise.

```lua
server.kick(user.id, "You have been removed from the server")
```

### Logging

#### `server.log(level, message)`

Writes a message to the server log. The log entry is tagged with
`source=plugin`.

| Level | Usage |
|-------|-------|
| `"debug"` | Verbose debugging information |
| `"info"` | Normal operational messages |
| `"warn"` | Warning conditions |
| `"error"` | Error conditions |

```lua
server.log("info", "Plugin loaded successfully")
server.log("warn", user.nick .. " triggered rate limit")
```

---

### Persistent Storage

Data saved with these functions survives server restarts and plugin reloads.
Each plugin has its own isolated data directory
(`Plugins/data/<plugin_name>/`).

#### `server.save(key, value)`

Saves a Lua value to disk as JSON.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | string | Storage key (alphanumeric, `-`, `_`, `.` only) |
| `value` | any | Lua value to save (string, number, boolean, table) |

**Key rules:**
- Must not be empty
- Allowed characters: `a-z`, `A-Z`, `0-9`, `-`, `_`, `.`
- Must not start with `.`
- Must not contain `..`

**Raises** a Lua error on invalid keys or write failures.

```lua
server.save("scores", {
    alice = 150,
    bob = 220,
    charlie = 95,
})
```

#### `server.load(key)`

Loads a previously saved value from disk.

**Returns:** The saved value (converted from JSON), or `nil` if the key
doesn't exist.

```lua
local scores = server.load("scores")
if scores then
    for name, score in pairs(scores) do
        server.log("info", name .. ": " .. score)
    end
end
```

**Type mapping (save/load):**

| Lua Type | JSON Type | Round-trips? |
|----------|-----------|:------------:|
| `string` | string | ✓ |
| `number` | number | ✓ |
| `boolean` | boolean | ✓ |
| `nil` | null | ✓ |
| table (array) | array | ✓ |
| table (map) | object | Keys become strings |

---

### HTTP Requests

Make outbound HTTP requests from plugins. Subject to the optional
allowlist/blocklist configured in `config.yaml`.

#### `server.http_get(url)`

Performs an HTTP GET request.

**Returns:** `status` (number), `body` (string)

- On success: HTTP status code and response body.
- On error: `0` and an error message string.

```lua
local status, body = server.http_get("https://api.example.com/data")
if status == 200 then
    server.log("info", "Got: " .. body)
else
    server.log("error", "HTTP error: " .. status .. " " .. body)
end
```

#### `server.http_post(url, body [, content_type])`

Performs an HTTP POST request.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `url` | string | *(required)* | Target URL |
| `body` | string | *(required)* | Request body |
| `content_type` | string | `"application/json"` | Content-Type header |

**Returns:** Same as `http_get()`.

```lua
local payload = '{"text": "Hello from Janus!"}'
local status, body = server.http_post(
    "https://hooks.slack.com/services/XXX",
    payload,
    "application/json"
)
```

**URL filtering:**
- Only `http://` and `https://` schemes are allowed. Other schemes (e.g.,
  `file://`, `ftp://`) are rejected.
- If `PluginConfig.HTTPAllowlist` is configured, the URL must match at least
  one pattern.
- If `PluginConfig.HTTPBlocklist` is configured, the URL must not match any
  pattern.
- Patterns are Go regular expressions matched against the full URL string.

---

### Timers

Schedule deferred or repeating callbacks. Timers run in the background and
acquire the plugin's Lua state lock when firing, so they are safe to use
alongside hooks.

#### `server.after(seconds, callback)`

Schedules a one-shot callback to fire after a delay.

**Returns:** Timer ID (number) for cancellation.

```lua
local id = server.after(60, function()
    server.send_chat("One minute has passed!")
end)
```

#### `server.every(seconds, callback)`

Schedules a repeating callback that fires at regular intervals.

**Returns:** Timer ID (number) for cancellation.

```lua
-- Announce the time every 15 minutes.
server.every(900, function()
    server.send_chat("Server time: " .. os.date("%H:%M"))
end)
```

#### `server.cancel_timer(id)`

Cancels a previously scheduled timer.

**Returns:** `true` if the timer was found and cancelled, `false` otherwise.

```lua
local timer_id = server.every(60, tick)

-- Later...
server.cancel_timer(timer_id)
```

**Notes:**
- Maximum **64 timers per plugin**. Exceeding this raises a Lua error.
- Timer callbacks have a 5-second execution timeout (same as hooks).
- All timers are automatically cancelled on plugin reload or server shutdown.

---

### Account & Config Access

Read-only access to account information and server configuration.

#### `server.get_account(login)`

Looks up an account by login name.

**Returns:** Account table, or `nil` if not found.

```lua
local acct = server.get_account("admin")
if acct then
    server.log("info", "Admin name: " .. acct.name)
    if acct.access.broadcast then
        server.log("info", "Admin can broadcast")
    end
end
```

**Account table fields:**

| Field | Type | Description |
|-------|------|-------------|
| `login` | string | Account login name |
| `name` | string | Display name |
| `access` | table | Permission flags (see below) |

**Access permission flags:**

Each flag is a boolean in the `access` sub-table:

| Flag | Description |
|------|-------------|
| `delete_file` | Can delete files |
| `upload_file` | Can upload files |
| `download_file` | Can download files |
| `rename_file` | Can rename files |
| `move_file` | Can move files |
| `create_folder` | Can create folders |
| `delete_folder` | Can delete folders |
| `rename_folder` | Can rename folders |
| `move_folder` | Can move folders |
| `read_chat` | Can read chat |
| `send_chat` | Can send chat |
| `create_chat` | Can create private chat rooms |
| `close_chat` | Can close private chat rooms |
| `show_in_list` | Visible in user list |
| `create_user` | Can create accounts |
| `delete_user` | Can delete accounts |
| `open_user` | Can view account details |
| `modify_user` | Can modify accounts |
| `read_news` | Can read news/message board |
| `post_news` | Can post to news/message board |
| `disconnect_user` | Can kick users |
| `cannot_be_disconnected` | Cannot be kicked |
| `get_client_info` | Can view client info |
| `upload_anywhere` | Can upload outside designated areas |
| `any_name` | Can use any nickname |
| `no_agreement` | Skips the server agreement |
| `set_file_comment` | Can set file comments |
| `set_folder_comment` | Can set folder comments |
| `view_drop_boxes` | Can view drop box folders |
| `make_alias` | Can create file aliases |
| `broadcast` | Can send server broadcasts |
| `read_msgboard` | Can read the message board |
| `write_msgboard` | Can write to the message board |
| `send_message` | Can send private messages |

#### `server.get_config()`

Returns a safe, read-only subset of the server configuration. Sensitive fields
(API keys, file paths, credentials) are not exposed.

**Returns:** Config table.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Server name |
| `description` | string | Server description |
| `max_downloads` | number | Global simultaneous download limit |
| `max_downloads_per_client` | number | Per-client download limit |
| `max_connections_per_ip` | number | Max connections per IP address |
| `disk_quota` | number | Server-wide disk quota (bytes) |
| `max_bandwidth` | number | Total bandwidth limit (bytes/sec) |
| `max_bandwidth_per_client` | number | Per-client bandwidth (bytes/sec) |
| `enable_tracker` | boolean | Tracker registration enabled |
| `allow_large_files` | boolean | 64-bit file support enabled |
| `enable_hope` | boolean | HOPE secure login enabled |
| `enable_chat_log` | boolean | Chat logging enabled |
| `enable_plugins` | boolean | Plugin system enabled |

```lua
local cfg = server.get_config()
server.send_chat("Welcome to " .. cfg.name)
```

---

### Inter-Plugin Communication

A shared key-value store that all loaded plugins can read from and write to.
Useful for coordinating between plugins or passing data across event
boundaries.

#### `server.set_shared(key, value)`

Sets a value in the shared store.

- Setting a key to `nil` deletes it.
- Values are copied — modifying a Lua table after sharing it does not affect
  the stored value.

```lua
server.set_shared("last_chatter", user.nick)
server.set_shared("online_count", #server.get_users())
```

#### `server.get_shared(key)`

Reads a value from the shared store.

**Returns:** The stored value, or `nil` if the key doesn't exist.

```lua
local last = server.get_shared("last_chatter")
if last then
    server.send_chat("Last chatter was: " .. last)
end
```

**Notes:**
- The shared store is thread-safe (backed by `sync.Map`).
- Shared state is **cleared on plugin reload** — use `server.save()`/`server.load()`
  for data that must survive reloads.
- Value types supported: strings, numbers, booleans, tables (arrays and maps).

---

### File System Access

Read-only access to the server's file tree. All paths are relative to the
server's `FileRoot` and **cannot escape it** — path traversal attempts
(`../`) are rejected.

#### `server.list_files([path])`

Lists the contents of a directory.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `path` | string | `"/"` | Directory path relative to FileRoot |

**Returns:** `files` (table or nil), `error` (string or nil)

Each entry in the returned array has:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | File or folder name |
| `is_dir` | boolean | `true` if the entry is a directory |
| `size` | number | File size in bytes, or child count for directories |
| `type` | string | File type code (e.g., `"TEXT"`, `"JPEG"`, `"fldr"`) |
| `modified` | number | Last modification time (Unix timestamp) |

```lua
local files, err = server.list_files("/")
if files then
    for _, f in ipairs(files) do
        local kind = f.is_dir and "dir" or "file"
        server.log("info", kind .. ": " .. f.name .. " (" .. f.size .. ")")
    end
else
    server.log("error", "list_files failed: " .. err)
end
```

#### `server.file_info(path)`

Gets detailed metadata for a single file or directory.

**Returns:** `info` (table or nil), `error` (string or nil)

Returns the same fields as `list_files` entries, plus:

| Field | Type | Description |
|-------|------|-------------|
| `comment` | string | File comment from metadata (empty if none) |

```lua
local info, err = server.file_info("readme.txt")
if info then
    server.send_message(user.id,
        info.name .. " — " .. info.size .. " bytes" ..
        (info.comment ~= "" and ("\nComment: " .. info.comment) or ""))
end
```

**Notes:**
- Hidden files, system files, and files matching `IgnoreFiles` patterns are
  not listed.
- Path traversal is blocked — attempts to access paths outside `FileRoot`
  return an error.

---

## Sandbox Environment

Plugins run in a restricted Lua 5.1 environment. The following standard
libraries are available:

| Library | Status | Notes |
|---------|--------|-------|
| `base` | Partial | `dofile` and `loadfile` removed |
| `table` | Full | |
| `string` | Full | |
| `math` | Full | |
| `os` | Partial | Only `time`, `difftime`, `clock`, `date` |
| `package` | Full | |

**Removed functions:** `dofile`, `loadfile`, `os.execute`, `os.exit`,
`os.remove`, `os.rename`, `os.tmpname`, `os.setlocale`, `os.getenv`

**Available `os` functions:**

```lua
os.time()              -- Current Unix timestamp
os.difftime(t2, t1)    -- Difference in seconds
os.clock()             -- CPU time used
os.date(format [, t])  -- Format a date string
```

---

## Limits & Constraints

| Limit | Value | Description |
|-------|-------|-------------|
| Hook timeout | 5 seconds | Maximum execution time per hook/timer call |
| Storage value size | 1 MiB | Maximum size per `server.save()` value |
| HTTP response body | 1 MiB | Maximum response body read by `http_get`/`http_post` |
| HTTP request timeout | 5 seconds | Per-request timeout for HTTP calls |
| Timers per plugin | 64 | Maximum concurrent timers per plugin |

Exceeding the hook timeout causes the Lua state's context to be cancelled,
which interrupts the running hook and logs an error. The plugin remains loaded
and can handle subsequent hooks normally.

---

## Configuration Reference

```yaml
# config.yaml

# Enable the plugin system.
EnablePlugins: true

# Directory containing .lua plugin files (relative to working directory).
PluginsDir: Plugins

# Optional URL filtering for server.http_get/http_post.
PluginConfig:
  # If set, only URLs matching at least one pattern are allowed.
  # Patterns are Go regular expressions matched against the full URL.
  HTTPAllowlist:
    - "^https://api\\.myservice\\.com/"
    - "^https://hooks\\.slack\\.com/"

  # URLs matching any pattern are blocked (checked after allowlist).
  HTTPBlocklist:
    - "^https?://localhost"
    - "^https?://127\\."
    - "^https?://10\\."
    - "^https?://192\\.168\\."
```

When both lists are empty (the default), all `http://` and `https://` URLs are
permitted. Non-HTTP schemes are always blocked.

---

## Best Practices

### State Management

- Use **local variables** at the top of your script for in-memory state. These
  persist for the lifetime of the plugin (until reload).
- Use **`server.save()`/`server.load()`** for data that must survive restarts.
- Use **`server.set_shared()`/`server.get_shared()`** for cross-plugin
  coordination (cleared on reload).

### Error Handling

- Wrap risky operations in `pcall()` to handle errors gracefully:

```lua
local ok, err = pcall(function()
    server.save("data", my_table)
end)
if not ok then
    server.log("error", "Failed to save: " .. tostring(err))
end
```

### Performance

- Pre-hooks should be fast — they block transaction processing.
- Avoid long-running loops in hooks. If you need background work, use timers.
- Be mindful of HTTP request timeouts (5s) in hooks — they count against the
  hook's 5-second timeout.

### Naming Conventions

- Prefix chat commands with `!` (e.g., `!help`, `!stats`).
- Use descriptive plugin filenames — the name is used in logs and data paths.
- Use alphabetic prefixes to control load order if it matters (e.g.,
  `00_core.lua`, `50_features.lua`).

### Security

- Never hardcode secrets in plugin files. Use `server.get_config()` for
  server-level settings.
- If your plugin makes HTTP requests, consider configuring the
  `HTTPAllowlist` to restrict outbound URLs to known-good endpoints.
- Use `server.get_account()` to verify permissions before executing
  sensitive operations.

---

## Examples

### Simple Greeter

```lua
function on_connect(user, ctx)
    server.send_message(user.id, "Welcome to the server, " .. user.nick .. "!")
    server.send_chat(user.nick .. " has joined the server.")
end

function on_disconnect(user)
    server.send_chat(user.nick .. " has left the server.")
end
```

### Word Filter (Pre-Hook Modification)

```lua
local replacements = {
    ["badword"] = "****",
    ["offensive"] = "********",
}

function pre_chat(user, ctx)
    local msg = ctx.message
    for word, replacement in pairs(replacements) do
        msg = msg:gsub(word, replacement)
    end
    return msg
end
```

### Scheduled Announcement

```lua
server.every(1800, function()
    server.send_chat("Remember to check the files section for new uploads!")
end)
```

### Persistent High Scores

```lua
local scores = server.load("scores") or {}

function on_chat(user, ctx)
    if ctx.message:sub(1, 6) == "!score" then
        local points = scores[user.login] or 0
        server.send_message(user.id, "Your score: " .. points)
    end
end

local function save_scores()
    server.save("scores", scores)
end
```

### Permission-Gated Command

```lua
function on_chat(user, ctx)
    if ctx.message == "!restart-warning" then
        local acct = server.get_account(user.login)
        if not acct or not acct.access.broadcast then
            server.send_message(user.id, "Permission denied.")
            return
        end
        server.broadcast("Server will restart shortly!")
    end
end
```

### Webhook Notification

```lua
function on_connect(user, ctx)
    local payload = '{"text": "' .. user.nick .. ' connected to the server"}'
    server.http_post("https://hooks.slack.com/services/XXX", payload)
end
```

### Cross-Plugin Communication

```lua
-- Plugin A: event_producer.lua
function on_upload(user, ctx)
    local count = (server.get_shared("upload_count") or 0) + 1
    server.set_shared("upload_count", count)
end

-- Plugin B: event_consumer.lua
function on_chat(user, ctx)
    if ctx.message == "!uploads" then
        local count = server.get_shared("upload_count") or 0
        server.send_message(user.id, "Total uploads this session: " .. count)
    end
end
```

### File Browser

```lua
function on_chat(user, ctx)
    local path = ctx.message:match("^!ls%s*(.*)")
    if path then
        if path == "" then path = "/" end
        local files, err = server.list_files(path)
        if not files then
            server.send_message(user.id, "Error: " .. err)
            return
        end
        local lines = {"Contents of " .. path .. ":"}
        for _, f in ipairs(files) do
            -- Icons to test UTF-8, old / unsupported clients will have some issues with these
            local icon = f.is_dir and "📁" or "📄"
            lines[#lines + 1] = "  " .. icon .. " " .. f.name ..
                " (" .. f.size .. (f.is_dir and " items" or " bytes") .. ")"
        end
        server.send_message(user.id, table.concat(lines, "\n"))
    end
end
```
