# Janus Plugin Developer Guide

> Last updated: September 9, 2026

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
   - [When hooks fire](#when-hooks-fire)
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
   - [Content Access](#content-access)
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

### Minimal Plugin

```lua
-- my_plugin.lua
function on_connect(user, ctx)
    server.send_message(user.id, "Welcome, " .. user.nick .. "!")
end
```

### Enabling Plugins

Add the following to `config.yaml`:

```yaml
EnablePlugins: true
PluginsDir: Plugins    # default
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

> **Hooks must be defined by the time the script finishes loading.** The server
> records which hook functions each plugin defines at load time and uses that
> to skip plugins that do not implement a given hook, so a hook assigned later
> — from a timer callback, or another hook — will never be called. Define every
> hook you might need at the top level and branch inside it:
>
> ```lua
> -- Works: the hook always exists, the config decides what it does.
> function pre_chat(user, ctx)
>     if not enabled then return end
>     -- ...
> end
>
> -- Does NOT work: assigned after load, so it is never dispatched.
> server.after(1, function() pre_chat = function(user, ctx) end end)
> ```

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

### When hooks fire

Both hook types fire around the handler, not around the *outcome*:

- **Pre-hooks run before any permission or validation check.** Access control
  lives inside each transaction handler, which has not run yet. A user with no
  upload rights still triggers `pre_upload`.
- **Post-hooks run whether the handler succeeded or failed.** A denied upload,
  a message to a user who has blocked the sender, a delete of a file that does
  not exist — each still fires its `on_*` hook. The hook is not told which
  happened.

This matters most for auditing and statistics plugins: counting `on_download`
counts download *requests*, not successful transfers. Where a completion signal
exists, use it — `on_upload_complete` fires only once the data is actually on
disk, unlike `on_upload`.

To act only on permitted operations, check the account yourself:

```lua
function on_delete_file(user, ctx)
    local acct = user.login ~= "" and server.get_account(user.login)
    if not (acct and acct.access.delete_file) then return end
    server.log("info", user.nick .. " deleted " .. ctx.filename)
end
```

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
| `upload` | `pre_upload` | `on_upload` | File upload requested |
| `upload_folder` | `pre_upload_folder` | `on_upload_folder` | Folder upload requested |
| *(upload complete)* | — | `on_upload_complete` | File data transfer finished |
| *(folder complete)* | — | `on_upload_folder_complete` | Folder data transfer finished |
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
| `msgboard_post` | `pre_msgboard_post` | `on_msgboard_post` | Message board post created |
| `news_post` | `pre_news_post` | `on_news_post` | News article posted |
| `news_delete` | `pre_news_delete` | `on_news_delete` | News article deleted |
| `news_category` | `pre_news_category` | `on_news_category` | News category created |
| `news_bundle` | `pre_news_bundle` | `on_news_bundle` | News bundle (folder) created |
| `news_item_delete` | `pre_news_item_delete` | `on_news_item_delete` | News item deleted |
| `voice_join` | `pre_voice_join` | `on_voice_join` | User joins a voice chat room |
| `voice_leave` | `pre_voice_leave` | `on_voice_leave` | User leaves a voice chat room |
| `voice_mute` | `pre_voice_mute` | `on_voice_mute` | User mutes or unmutes in voice chat |
| `im` | `pre_im` | `on_im` | Messaging-extension instant message sent |
| `friend_request` | `pre_friend_request` | `on_friend_request` | User adds someone to their roster |
| `presence` | `pre_presence` | `on_presence` | User changes presence state or status text |
| `call_invite` | `pre_call_invite` | `on_call_invite` | User invites others to a call |
| *(disconnect)* | — | `on_disconnect` | User disconnects (no `ctx`) |
| *(msgboard delete)* | — | `on_msgboard_delete` | Post deleted via REST API (no `pre_` variant) |

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
| `filepath` | string | Directory path (e.g. `"/subfolder"`) — present when upload targets a subdirectory |

> **Note:** `on_upload` fires on the upload *request*, before any data
> transfers and regardless of whether the request was permitted. Use
> `on_upload_complete` to react once the file is actually on disk.

### Upload/Folder Complete (`on_upload_complete` / `on_upload_folder_complete`)

These content hooks fire after the file data transfer finishes. They receive
only a `ctx` table (no `user`).

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Full path relative to file root (e.g. `"/subfolder/photo.jpg"`) |

### Disconnect User (`pre_disconnect_user` / `on_disconnect_user`)

| Field | Type | Description |
|-------|------|-------------|
| `target_id` | number | ID of the user being disconnected |

### User Change (`pre_user_change` / `on_user_change`)

| Field | Type | Description |
|-------|------|-------------|
| `nick` | string | New nickname (if changed) |
| `icon` | number | New icon ID (if changed) |

### Message Board Post (`pre_msgboard_post` / `on_msgboard_post`)

| Field | Type | Description |
|-------|------|-------------|
| `body` | string | The post body text |

### News Article Post (`pre_news_post` / `on_news_post`)

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Slash-delimited category path |
| `title` | string | Article title |
| `body` | string | Article body text |
| `parent_id` | number | Parent article ID (0 for top-level) |

### News Article Delete (`pre_news_delete` / `on_news_delete`)

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Slash-delimited category path |
| `article_id` | number | ID of the deleted article |

### News Category (`pre_news_category` / `on_news_category`)

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Slash-delimited parent category path |
| `name` | string | Name of the new category |

### News Bundle (`pre_news_bundle` / `on_news_bundle`)

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Slash-delimited parent category path |
| `name` | string | Name of the new bundle |

### News Item Delete (`pre_news_item_delete` / `on_news_item_delete`)

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Slash-delimited path of the deleted item |

### Voice Chat (`pre_voice_join` / `on_voice_join`, etc.)

The `voice_join`, `voice_leave`, and `voice_mute` hooks fire on voice chat
activity. The `ctx` table has no additional fields — the `user` table
identifies the participant.

```lua
function on_voice_join(user, ctx)
    server.send_chat(user.nick .. " joined voice chat")
end
```

### Messaging Extension

These hooks cover the messaging extension (instant messages, rosters, presence
and calls) rather than classic Hotline chat. In practice only clients speaking
the extension send these transactions — but the hooks fire on the *attempt*,
before the server checks that messaging is enabled for the sender, so do not
treat `on_im` as proof a message was delivered (see
[When hooks fire](#when-hooks-fire)).

#### Instant Message (`pre_im` / `on_im`)

| Field | Type | Description |
|-------|------|-------------|
| `recipient` | string | Recipient's account login |
| `message` | string | The message body |

`pre_im` is the natural place for anti-spam — returning `false` drops the
message before delivery. See `im_anti_spam.lua` in the bundled plugins.

#### Friend Request (`pre_friend_request` / `on_friend_request`)

| Field | Type | Description |
|-------|------|-------------|
| `target` | string | Login of the account being added |

#### Presence (`pre_presence` / `on_presence`)

| Field | Type | Description |
|-------|------|-------------|
| `state` | number | Presence state (see below) |
| `status_text` | string | Free-text status, if the client set one |

| `state` | Meaning |
|---------|---------|
| `1` | Online / available |
| `2` | Away |
| `3` | Invisible |
| `4` | Busy (do not disturb) |

#### Call Invite (`pre_call_invite` / `on_call_invite`)

| Field | Type | Description |
|-------|------|-------------|
| `invitees` | table | Array of account logins invited to the call |

```lua
function on_call_invite(user, ctx)
    server.log("info", user.nick .. " invited " .. #ctx.invitees .. " people to a call")
end
```

### Message Board Delete (`on_msgboard_delete`)

This is an API-only hook fired via `FireContentHook` when a post is deleted
through `DELETE /api/v2/msgboard/{id}`. It has **no pre-hook variant** and
receives only a `ctx` table (no `user`).

| Field | Type | Description |
|-------|------|-------------|
| `post_id` | number | ID of the deleted post |

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

The disconnect is deferred briefly so the message can reach the client first.
At most 64 such kicks may be pending at once; beyond that the disconnect is
immediate and the message may not be seen. Pending kicks are released on
server shutdown.

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
- At most 200 characters
- Allowed characters: `a-z`, `A-Z`, `0-9`, `-`, `_`, `.`
- Must not start with `.`
- Must not contain `..`
- Case-insensitive (see below)

**Raises** a Lua error on invalid keys or write failures.

```lua
server.save("scores", {
    alice = 150,
    bob = 220,
    charlie = 95,
})
```

> **Keys are case-insensitive.** They map directly to filenames, so `"Scores"`
> and `"scores"` address the same value on every platform — without this they
> would collide on macOS and Windows but not on Linux, and a plugin developed on
> one and deployed on the other would silently lose data. Values written by an
> older build under a mixed-case name are migrated on the next save.

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
| table (mixed) | array | **String keys are lost** — see below |

> **Mixed tables lose their string keys.** A table with both sequential integer
> keys and string keys is serialised as an *array*, and the string-keyed entries
> are dropped without an error. Keep the two shapes separate:
>
> ```lua
> -- Loses "total": saved as ["a", "b"]
> server.save("bad", { "a", "b", total = 2 })
>
> -- Keeps everything
> server.save("good", { items = { "a", "b" }, total = 2 })
> ```
>
> The same conversion is used by `server.set_shared()`, so the caveat applies
> there too.

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
- Redirects are followed (up to 10 hops), and **each redirect target is
  re-checked** against the allowlist and blocklist.
- Connections to loopback, private, link-local and unspecified addresses are
  refused. The check runs at connection time against the resolved address, so
  a hostname cannot resolve to a permitted address for the check and a private
  one for the connection.

**Timeouts:** a request is bound to the context of the hook or timer that
issued it, so it is cancelled when that budget expires — 5 seconds inside a
hook, 60 seconds inside a timer callback. A 30-second ceiling applies on top
of whichever is shorter.

> **Do not make HTTP requests from a pre-hook on a busy server.** The request
> holds the plugin's Lua state for its duration, and every other user's hooks
> for that plugin queue behind it. Prefer a timer that fetches periodically and
> caches the result with `server.save` or `server.set_shared`.

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
- Timer callbacks have a **60-second** execution timeout (hooks get 5).
- All timers are automatically cancelled on plugin reload or server shutdown.
- **Timers are not a concurrency escape hatch.** A callback runs on its own
  goroutine but acquires the same Lua state lock as your hooks, so slow work in
  a timer blocks every hook in that plugin — for all users — while it runs.
  Timers are for work that happens *on a schedule*, not for moving expensive
  work off the request path.

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

### Password Hashing

Native bcrypt hashing for plugins that need to store their own passwords or
secrets (e.g. a nickname-registration service). The sandbox has no crypto
library of its own, so these wrap the same `bcrypt` implementation Janus uses
for Hotline account passwords — there's no need to hand-roll a hash in Lua.

#### `server.bcrypt_hash(password)`

Hashes a password with bcrypt. The salt and cost factor are generated
internally and embedded in the returned hash string, so nothing else needs
to be stored alongside it.

**Returns:** `hash, err` — `hash` is `nil` if hashing failed (for example, a
password longer than 72 bytes), in which case `err` describes why.

```lua
local hash, err = server.bcrypt_hash(password)
if not hash then
    server.log("error", "hash failed: " .. tostring(err))
    return
end
```

#### `server.bcrypt_verify(password, hash)`

Checks a plaintext password against a previously stored bcrypt hash.

**Returns:** `true` if the password matches, `false` otherwise.

```lua
if server.bcrypt_verify(password, stored_hash) then
    server.log("info", "password verified")
end
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
| `creator` | string | Creator signature from metadata (empty if none) |
| `comment` | string | File comment from metadata (empty if none) |

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
| `download_count` | number | Times the file has been downloaded (0 if untracked) |

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

### Content Access

Read-only access to message board posts, news categories, news articles,
folder type detection, and disk catalog queries. These functions are designed
for content indexing and synchronization plugins.

#### `server.file_access(path)`

Detects the type of a folder based on naming conventions.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | string | Path relative to FileRoot |

**Returns:** One of `"normal"`, `"drop_box"`, or `"upload"`.

Detection is based on the folder name (case-insensitive):
- Names containing `"drop box"` or `"dropbox"` → `"drop_box"`
- Names containing `"upload"` → `"upload"`
- Everything else → `"normal"`

Non-directory paths and errors return `"normal"`.

```lua
local access = server.file_access("Uploads")
if access == "drop_box" then
    server.log("info", "This is a drop box folder")
end
```

#### `server.search_files(opts)`

Queries the disk catalog for files matching the given criteria.

**Parameters:**

The single argument is a table with optional filter keys:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `mod_after` | number | `0` | Unix timestamp — only return files modified after this time |
| `limit` | number | `0` | Maximum results (0 = no limit) |

**Returns:** `files` (table or nil), `error` (string or nil)

Each entry has the same fields as `list_files`, plus:

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Path relative to FileRoot |
| `download_count` | number | Times the file has been downloaded |

```lua
-- Find files modified in the last hour.
local files, err = server.search_files({
    mod_after = os.time() - 3600,
    limit = 50,
})
if files then
    for _, f in ipairs(files) do
        server.log("info", "Recent: " .. f.name .. " (" .. f.size .. " bytes)")
    end
end
```

#### `server.recent_files([limit])`

Returns the most recently modified files from the disk catalog.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | `25` | Maximum number of files to return |

**Returns:** `files` (table or nil), `error` (string or nil)

Each entry has the same fields as `search_files`, except that `is_dir` is not
set — the disk catalog's recent list contains files only.

```lua
local recent, err = server.recent_files(10)
if recent then
    for _, f in ipairs(recent) do
        server.send_chat("New file: " .. f.name)
    end
end
```

#### `server.storage_stats()`

Returns aggregate statistics about the server's file tree from the disk
catalog.

**Returns:** `stats` (table or nil), `error` (string or nil)

| Field | Type | Description |
|-------|------|-------------|
| `total_files` | number | Total file count |
| `total_dirs` | number | Total directory count |
| `total_bytes` | number | Total size of all files in bytes |
| `top_folders` | table | Array of top-level folders with `name`, `total_files`, `total_dirs`, `total_bytes` |
| `type_breakdown` | table | Array of type entries with `type_code`, `file_count`, `total_bytes` |
| `largest_files` | table | Array of largest files with `name`, `path`, `size`, `type` |
| `most_downloaded` | table | Array of most downloaded files with `name`, `path`, `size`, `download_count` |

```lua
local stats, err = server.storage_stats()
if stats then
    server.send_chat(string.format(
        "Files: %d, Dirs: %d, Total: %.1f MB",
        stats.total_files, stats.total_dirs,
        stats.total_bytes / 1048576
    ))
end
```

#### `server.list_msgboard_posts([limit, offset])`

Returns a paginated list of message board posts.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | `50` | Maximum number of posts to return |
| `offset` | number | `0` | Number of posts to skip |

**Returns:** Array of post tables. Returns an empty table on error.

Each post table has:

| Field | Type | Description |
|-------|------|-------------|
| `id` | number | Post ID |
| `login` | string | Account login of the poster |
| `nick` | string | Display name of the poster |
| `body` | string | Post body text |
| `timestamp` | string | ISO 8601 timestamp (RFC 3339) |

```lua
local posts = server.list_msgboard_posts(10, 0)
for _, p in ipairs(posts) do
    server.log("info", p.nick .. ": " .. p.body)
end
```

#### `server.get_msgboard_count()`

Returns the total number of message board posts.

**Returns:** Number (integer). Returns `0` on error.

```lua
local count = server.get_msgboard_count()
server.log("info", "Message board has " .. count .. " posts")
```

#### `server.list_news_categories([path])`

Lists news categories and bundles at the given path.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `path` | string | `"/"` | Slash-delimited category path |

**Returns:** Array of category tables. Returns an empty table on error.

Each category table has:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Category or bundle name |
| `kind` | string | `"category"` or `"bundle"` |

```lua
local cats = server.list_news_categories("/")
for _, c in ipairs(cats) do
    server.log("info", c.kind .. ": " .. c.name)
end
```

#### `server.list_news_articles(path [, limit, offset])`

Returns a paginated list of news articles in a category.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `path` | string | *(required)* | Slash-delimited category path |
| `limit` | number | `50` | Maximum number of articles to return |
| `offset` | number | `0` | Number of articles to skip |

**Returns:** Array of article tables. Returns an empty table on error.

Each article table has:

| Field | Type | Description |
|-------|------|-------------|
| `id` | number | Article ID |
| `title` | string | Article title |
| `poster` | string | Author name |
| `date` | string | ISO 8601 timestamp (RFC 3339) |
| `body` | string | Article body text |
| `parent_id` | number | Parent article ID (0 for top-level) |

```lua
local articles = server.list_news_articles("General/Announcements", 20, 0)
for _, a in ipairs(articles) do
    server.log("info", a.title .. " by " .. a.poster)
end
```

#### `server.get_news_article_count([path])`

Returns the total number of articles in a news category.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `path` | string | `"/"` | Slash-delimited category path |

**Returns:** Number (integer). Returns `0` on error.

```lua
local count = server.get_news_article_count("General/Announcements")
server.log("info", "Category has " .. count .. " articles")
```

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
| `package` | Full | `require` works, but there is no `io` to load data with |
| `io` | **Absent** | Not opened — no file or stream access |
| `debug` | **Absent** | Not opened |
| `coroutine` | **Absent** | Not opened — `coroutine.create` and friends are nil |

**Removed functions:** `dofile`, `loadfile`, `os.execute`, `os.exit`,
`os.remove`, `os.rename`, `os.tmpname`, `os.setlocale`, `os.getenv`

**Still present, and worth knowing about:**

- `load` and `loadstring` are available, so a plugin can compile code from a
  string. Plugins are operator-installed and trusted, so this is not a sandbox
  hole, but do not use it to run text that reached the server from a user.
- `print` writes to the server's standard output, bypassing the log entirely —
  it will not appear in the server log or be captured by a service manager's
  log handling. Use `server.log()` instead.

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
| Hook timeout | 5 seconds | Maximum execution time per hook call |
| Timer timeout | 60 seconds | Maximum execution time per timer callback |
| Storage value size | 1 MiB | Maximum size per `server.save()` value |
| HTTP response body | 1 MiB | Maximum response body read by `http_get`/`http_post` |
| HTTP request timeout | hook/timer budget | Bound to the calling hook (5s) or timer (60s), with a 30s ceiling |
| Timers per plugin | 64 | Maximum concurrent timers per plugin |
| Consecutive hook errors | 10 | A hook failing this many times in a row is disabled until the next reload |
| Pending kicks | 64 | Deferred `server.kick` calls in flight; past this the kick is immediate |

Exceeding the hook timeout causes the Lua state's context to be cancelled,
which interrupts the running hook and logs an error. The plugin remains loaded
and can handle subsequent hooks normally.

**Repeated failures disable a hook.** If a hook raises an error on 10
consecutive calls it is switched off for that plugin and a single warning is
logged, rather than an error on every transaction from then on. Any successful
call resets the count, so an intermittently failing hook is never disabled.
Reloading plugins re-enables everything.

**What the timeout does and does not bound.** Cancellation is observed by the
Lua VM between instructions, so it interrupts *Lua* execution promptly. It
cannot interrupt a Go function that is already running — the filesystem and
content builtins (`server.search_files`, `server.storage_stats`,
`server.list_news_articles`, …) run to completion regardless, and they hold the
plugin's state lock while they do. HTTP calls are the exception: they are bound
to the same context and are cancelled with it.

Keep hooks short. A hook that takes 200 ms makes that operation take 200 ms for
every user of the server, not just the one who triggered it.

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
  # Note there is no need to list private address ranges here — they are
  # refused at connection time regardless of configuration.
  HTTPBlocklist:
    - "^https?://internal\\.example\\.com/"
```

When both lists are empty (the default), any `http://` or `https://` URL is
permitted **except** those that resolve to a loopback, private, link-local or
unspecified address; those are always refused, at the point the connection is
made rather than by inspecting the URL, so a hostname cannot resolve to a
public address for the check and a private one for the request. Redirect
targets are re-checked against both lists. Non-HTTP schemes are always blocked.

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

- Pre-hooks should be fast — they block transaction processing. A hook that
  takes 200 ms makes that operation take 200 ms for *every* user, not just the
  one who triggered it.
- Avoid long-running loops in hooks. Timers are for work that happens on a
  schedule, **not** a way to move slow work off the request path — a timer
  callback takes the same plugin state lock your hooks do, so it blocks them
  while it runs.
- HTTP requests are bound to the calling hook's budget (5 s) or timer's (60 s),
  with a 30 s ceiling. Prefer fetching on a timer and caching the result with
  `server.save()` or `server.set_shared()` over fetching inside a hook.
- The filesystem and content builtins (`server.search_files`,
  `server.storage_stats`, `server.list_news_articles`, …) are *not* interrupted
  by the hook timeout and hold the state lock for their whole duration. Call
  them from timers with a cached result, not from a chat hook.

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
function on_upload_complete(ctx)
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
            local icon = f.is_dir and "📁" or "📄"
            lines[#lines + 1] = "  " .. icon .. " " .. f.name ..
                " (" .. f.size .. (f.is_dir and " items" or " bytes") .. ")"
        end
        server.send_message(user.id, table.concat(lines, "\n"))
    end
end
```
