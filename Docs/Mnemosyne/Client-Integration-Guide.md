# Mnemosyne Search – Client Integration Guide

**Specification for implementing content search in a Hotline client via the Mnemosyne index API**

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Configuration](#configuration)
- [Authentication](#authentication)
- [Rate Limits](#rate-limits)
- [Error Handling](#error-handling)
- [Conventions](#conventions)
- [API Reference](#api-reference)
  - [Health Check](#health-check)
  - [Search](#search)
  - [List Servers](#list-servers)
  - [Server Detail](#server-detail)
  - [Browse News](#browse-news)
  - [Browse Files](#browse-files)
  - [Statistics](#statistics)
- [Connecting to a Result](#connecting-to-a-result)
- [Recommended UI Flow](#recommended-ui-flow)
- [Implementation Notes](#implementation-notes)

---

## Overview

Mnemosyne is an optional indexing service for the Hotline ecosystem. Hotline servers that opt in sync their content (message board posts, news articles, and file listings) to a Mnemosyne instance, which provides a full-text search API over the aggregated index.

**Do note** that the content can be a constellation of different configurations, such as file listings only, or file listings and news articles, or only news articles etc.

Client developers can integrate Mnemosyne search to let users discover servers and content **before** connecting. This is particularly useful for:

- Finding servers that host specific files or topics.
- Searching message board posts across multiple servers.
- Browsing news articles from the broader Hotline community.
- Previewing a server's content before committing to a connection.

Mnemosyne search is purely read-only and does not require authentication.

---

## Architecture

```
┌───────────┐           ┌───────────┐           ┌──────────────┐
│  Client   │--HTTP---> │ Mnemosyne │ <---sync--│ Hotline      │
│  (yours)  │ <---GET-- │  Index    │           │ Servers      │
└───────────┘           └───────────┘           └──────────────┘
```

Your client communicates only with the Mnemosyne HTTP API. The sync between Hotline servers and Mnemosyne happens independently, server-side, and is not visible to clients.

Only **public** servers appear anywhere in the API: servers that have completed a first sync (status `active` or `stale`) and whose operator account is active. A server that is not public is reported as not found (`404`), exactly as if it did not exist.

---

## Configuration

Your client needs a single user-configurable setting:

| Setting | Description | Example |
|---------|-------------|---------|
| Mnemosyne URL | Base URL of the Mnemosyne instance | `http://mnemosyne.example.com:8980` |

Store this in your client's preferences/settings. There is no registration or account creation required. If the URL is empty or unset, disable search functionality gracefully.

A web service or proxy that queries Mnemosyne on behalf of many users (like Agora) should also have a **read key** setting; see [Rate Limits](#rate-limits).

---

## Authentication

All read endpoints are **public** - no API key is required.

If you do provide a key, it can be passed via:

- **Header:** `X-API-Key: <key>` (preferred)
- **Query parameter:** `?api_key=<key>` - **read keys only** (they start with `mrd_`). Any other kind of key in a URL is refused with `401`, since URLs end up in proxy and server logs.

Invalid keys are rejected with `401`. Omitting the key entirely is always accepted.

The practical reason to use a key is rate limiting: requests with a key are counted per key instead of per address.

---

## Rate Limits

Mnemosyne enforces rate limits on the read endpoints. The defaults (an instance may configure others) are:

| Requests | Limit |
|----------|-------|
| Without a key | **120 per minute per client address** |
| With a key | **1200 per minute per key** |

If exceeded, the server returns `429 Too Many Requests` with a `Retry-After` header giving the number of seconds to wait.

Clients should:
- Debounce search input (avoid firing a request on every keystroke).
- Cache results locally when practical.
- Respect `429` responses: wait at least `Retry-After` seconds before retrying.

A service that queries Mnemosyne for many users from one address (a website, a bridge, a bot) would otherwise share one per-address budget among all of them. Ask the Mnemosyne operator for a read key and send it with every request.

---

## Error Handling

All errors follow a consistent JSON envelope:

```json
{
  "error": {
    "code": "bad_request",
    "message": "human-readable description"
  }
}
```

Common HTTP status codes:

| Status | Meaning |
|--------|---------|
| `200` | Success |
| `400` | Bad request (missing or invalid parameters, query too long) |
| `401` | Invalid API key (only if one was provided), or a non-read key in `?api_key=` |
| `404` | Resource not found (unknown or non-public `server_id`) |
| `429` | Rate limit exceeded (see `Retry-After`) |
| `500` | Internal server error |
| `503` | Service degraded (health check only) |

Branch on the HTTP status and `error.code`; the `message` is meant for humans and may change.

---

## Conventions

- **Server IDs** are short identifiers chosen by the server's operator when registering it: lowercase letters, digits, `-` and `_`, 3–64 characters (e.g. `retro-hub`). Treat them as opaque strings.
- **Timestamps** are RFC 3339 in UTC, to the second (e.g. `2026-03-30T10:00:00Z`). Content timestamps (`timestamp`, `date`) are as the source server reported them, converted to UTC. A value the server sent in a format Mnemosyne doesn't recognize is passed through as sent, and any of them may be an empty string when the server sent none. Parse defensively.
- **Paths** use `/` as the separator and start with `/`. A file's `path` is its full path including its name. Folder paths have no trailing slash. Paths, names and text are UTF-8.
- **Arrays** in responses are always present, empty (`[]`) when there is nothing to list.

---

## API Reference

All endpoints are prefixed with `/api/v1`. Responses are `application/json`.

---

### Health Check

```
GET /api/v1/health
```

Use this to verify the Mnemosyne instance is reachable before enabling search features. The health check is not rate limited.

**Response:**
```json
{
  "status": "ok",
  "version": "1.2.3",
  "uptime_seconds": 3600,
  "database": "ok"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"ok"` or `"degraded"` |
| `version` | string | Mnemosyne software version |
| `uptime_seconds` | int | Server uptime |
| `database` | string | `"ok"` or `"degraded"` |

Returns `503` when `status` is `"degraded"`.

---

### Search

```
GET /api/v1/search?q=<query>
```

Full-text search across all indexed content. This is the primary endpoint most clients will use.

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `q` | string | **yes** | - | Search query text (at most 256 bytes) |
| `type` | string | no | all | Filter by content type: `msgboard`, `news`, or `files` |
| `server` | string | no | all | Filter by `server_id` |
| `limit` | int | no | `20` | Maximum results to return (capped by the instance, `100` by default) |
| `offset` | int | no | `0` | Pagination offset |

**Query syntax:**

- The query is split into words; a result must match **every** word (case-insensitive). Up to 16 words are used.
- Words are matched literally: quotes, `AND`/`OR`/`NOT`, `:` and other search-engine operators have no special meaning.
- A word ending in `*` matches as a **prefix**, if at least two characters come before the `*`: `resed*` finds `ResEdit`. Without the `*`, words match whole words only (`resed` does not find `ResEdit`).
- Message board posts match on nick and body; news articles on title, poster and body; files on name, comment and path (a file name match ranks highest).
- A query with no searchable words left (e.g. only quotes) returns no results rather than an error.

**Response:**
```json
{
  "total": 57,
  "total_capped": false,
  "results": [
    {
      "type": "msgboard",
      "server_id": "retro-hub",
      "server_name": "My Server",
      "server_address": "example.com:5500",
      "score": 12.5,
      "data": {
        "post_id": 42,
        "nick": "admin",
        "body": "This is a truncated snippet of the post...",
        "timestamp": "2026-03-30T10:00:00Z"
      }
    },
    {
      "type": "news",
      "server_id": "retro-hub",
      "server_name": "My Server",
      "server_address": "example.com:5500",
      "score": 8.2,
      "data": {
        "path": "/News/General",
        "article_id": 5,
        "title": "Article Title",
        "poster": "admin",
        "body": "This is a truncated snippet of the article...",
        "date": "2026-03-28T07:00:00Z"
      }
    },
    {
      "type": "file",
      "server_id": "retro-hub",
      "server_name": "My Server",
      "server_address": "example.com:5500",
      "score": 5.0,
      "data": {
        "path": "/Images/photo.jpg",
        "name": "photo.jpg",
        "size": 204800,
        "type": "JPEG",
        "comment": "A nice photo"
      }
    }
  ]
}
```

**Top-level fields:**

| Field | Type | Description |
|-------|------|-------------|
| `total` | int | Total matching results across the requested types (may exceed `limit`) |
| `total_capped` | bool | `true` if a content type had more than 10,000 matches; it then contributes only 10,000 to `total`. Show e.g. "10,000+ results" |
| `results` | array | The requested page of results |

**Result ordering:** results are grouped by type: all message board matches first, then news, then files. Within each group they are ordered by relevance. `offset` and `limit` page through that combined list, so a page can span two groups. If your UI should show types separately (tabs, sections), make one request per `type` instead.

**Result fields:**

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Content type: `"msgboard"`, `"news"`, or `"file"` (note: `file`, while the filter parameter is `files`) |
| `server_id` | string | ID of the source server |
| `server_name` | string | Display name of the source server |
| `server_address` | string | Hotline address of the source server (`host:port`) |
| `score` | float | Relevance score (higher = more relevant). Only comparable between results of the same type in the same response; the scale depends on the size of the index |
| `data` | object | Type-specific fields (see below) |

**Data fields by type:**

Message board (`msgboard`):

| Field | Type | Description |
|-------|------|-------------|
| `post_id` | int | Post identifier |
| `nick` | string | Author nickname |
| `body` | string | Post body (truncated; see [Body truncation](#implementation-notes)) |
| `timestamp` | string | Post timestamp |

News article (`news`):

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | News category path (e.g. `"/News/General"`) |
| `article_id` | int | Article identifier (unique within its category) |
| `title` | string | Article title |
| `poster` | string | Author name |
| `body` | string | Article body (truncated) |
| `date` | string | Article date |

File (`file`):

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Full path of the file, including its name (e.g. `"/Images/photo.jpg"`) |
| `name` | string | File name |
| `size` | int | File size in bytes |
| `type` | string | Hotline type code (e.g. `"JPEG"`, `"TEXT"`; may be empty) |
| `comment` | string | File comment (may be empty) |

---

### List Servers

```
GET /api/v1/servers
```

Lists all public servers in the Mnemosyne index (see [Architecture](#architecture)).

**Query Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `address` | string | no | Filter by exact server address |

**Response:**
```json
{
  "servers": [
    {
      "server_id": "retro-hub",
      "server_name": "My Server",
      "server_address": "example.com:5500",
      "last_sync": "2026-03-31T12:00:00Z",
      "counts": {
        "msgboard_posts": 42,
        "news_articles": 15,
        "files": 200,
        "total_file_size": 1048576
      }
    }
  ]
}
```

Sorted by `last_sync` descending. The `last_sync` field is omitted if the server has never synced. The counts are of indexed content (folders are not counted as files), so they match what search can find.

---

### Server Detail

```
GET /api/v1/servers/{server_id}
```

Detailed view of a single server, including recent content samples. Useful for showing a server preview before the user connects.

**Response:**
```json
{
  "server_id": "retro-hub",
  "server_name": "My Server",
  "server_address": "example.com:5500",
  "server_description": "Welcome to my server!",
  "status": "active",
  "last_sync": "2026-03-31T12:00:00Z",
  "last_heartbeat": "2026-03-31T12:05:00Z",
  "counts": {
    "msgboard_posts": 42,
    "news_articles": 15,
    "files": 200,
    "total_file_size": 1048576
  },
  "recent_posts": [
    {"id": 10, "nick": "admin", "body": "Latest post preview...", "timestamp": "2026-03-31T11:58:00Z"}
  ],
  "recent_articles": [
    {"id": 5, "path": "/News", "title": "Latest Update", "poster": "admin", "date": "2026-03-30T09:00:00Z"}
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"active"`, or `"stale"` if the server hasn't been heard from for a while (its index may be out of date) |
| `last_sync`, `last_heartbeat` | string | Omitted if never |
| `recent_posts` | array | Up to 5 most recent message board posts (body truncated) |
| `recent_articles` | array | Up to 5 most recent news articles (title/poster only, no body) |

Returns `404` if the `server_id` is unknown or not public.

---

### Browse News

```
GET /api/v1/servers/{server_id}/news?path=/
```

Browse the news category hierarchy for a specific server, one level at a time. Provides full article bodies (not truncated).

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `path` | string | no | `"/"` | News category path to browse (a trailing `/` is ignored) |
| `limit` | int | no | `100` | Articles per page (at most `1000`) |
| `offset` | int | no | `0` | Article pagination offset |

**Response:**
```json
{
  "path": "/",
  "categories": [
    {"path": "/Announcements", "name": "Announcements"},
    {"path": "/General", "name": "General"}
  ],
  "articles": [
    {
      "article_id": 1,
      "title": "Welcome",
      "poster": "admin",
      "date": "2026-03-01T12:00:00Z",
      "body": "Full article body text (not truncated)"
    }
  ],
  "total_articles": 1
}
```

- `categories`: direct child categories of the requested path, sorted by name. All of them are returned (no paging).
- `articles`: the articles in the category at exactly `path`, newest first (by `article_id` descending), paged by `limit`/`offset`.
- `total_articles`: how many articles that category holds; page until `offset >= total_articles`.
- `date` is omitted when the server sent none.
- Returns `404` if the `server_id` is unknown or not public. An unknown `path` returns empty arrays.

---

### Browse Files

```
GET /api/v1/servers/{server_id}/files?path=/
```

Browse the file tree for a specific server, one folder at a time.

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `path` | string | no | `"/"` | Folder path to browse (a trailing `/` is ignored) |
| `limit` | int | no | `500` | Entries per page (at most `5000`) |
| `offset` | int | no | `0` | Pagination offset |

**Response:**
```json
{
  "path": "/",
  "files": [
    {
      "path": "/Images",
      "name": "Images",
      "size": 0,
      "is_dir": true
    },
    {
      "path": "/readme.txt",
      "name": "readme.txt",
      "size": 1024,
      "type": "TEXT",
      "creator": "ttxt",
      "comment": "Read me first",
      "is_dir": false
    }
  ],
  "total": 2
}
```

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Full path of the entry (folders: no trailing slash). Pass a folder's `path` as the next `path` parameter to open it |
| `name` | string | Entry name |
| `is_dir` | bool | `true` for folders, `false` for files |
| `size` | int | File size in bytes (`0` for folders) |
| `type` | string | Hotline 4-char type code - **omitted** when empty |
| `creator` | string | Hotline 4-char creator code - **omitted** when empty |
| `comment` | string | File comment - **omitted** when empty |
| `total` | int | Number of entries (folders and files) in this folder |

Folders come first, then files, each sorted by name (byte order, so case-sensitive: `Zebra` sorts before `apple`); sort again client-side if you prefer another order. `limit`/`offset` page through that combined list.

Folders are listed even when the server only reported the files inside them (many servers send files only), so you can always navigate down to every indexed file.

Returns `404` if the `server_id` is unknown or not public. An unknown `path` returns an empty `files` array.

---

### Statistics

```
GET /api/v1/stats
```

Aggregate statistics across the public part of the Mnemosyne index. Useful for an "about" or status display. The figures may be up to 30 seconds old.

**Response:**
```json
{
  "servers": {
    "total": 25,
    "active": 20
  },
  "content": {
    "msgboard_posts": 1500,
    "news_articles": 800,
    "files": 5000,
    "total_file_size": 10737418240
  },
  "most_active": [
    {"server_id": "big-server", "server_name": "Big Server", "total_content": 500}
  ],
  "recently_updated": [
    {"server_id": "fresh-server", "server_name": "Fresh Server", "last_sync": "2026-03-31T12:00:00Z"}
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `servers.total` | int | Public servers (active or stale) |
| `servers.active` | int | Of those, the ones currently active |
| `most_active` | array | Top 10 servers by total content count |
| `recently_updated` | array | Top 10 most recently synced servers |
| `total_file_size` | int | Aggregate file size in bytes |

---

## Connecting to a Result

Every search result includes a `server_address` field in `host:port` format (e.g. `"example.com:5500"`). This is a standard Hotline server address that can be used directly to initiate a connection.

When a user selects a search result:

1. Extract `server_address` from the result.
2. Use it as the target for a standard Hotline TCP connection (default port `5500` if not specified).
3. Optionally pre-fill the server name from `server_name` for display during connection.
4. Connect as a guest by default - the client has no way to know what credentials are required.

The address can contain a hostname or IP, with or without a port number. Treat it identically to how your client handles manually-entered server addresses.

The address is the one the server's operator registered; the server may be offline (a `stale` server in particular). Handle a failed connection as you would for any manually entered address.

To take the user to the content itself once connected, use the result's `path` (files, news) as a starting point for the client's file or news browser. Indexes can lag the live server, so the item may have moved or gone.

---

## Recommended UI Flow

A typical integration provides three views:

### 1. Search Input

- A text field for the search query.
- An optional type filter toggle (all / files / news / msgboard).
- Fire the search request when the user presses Enter (not on every keystroke - see [Rate Limits](#rate-limits)).
- Mention that `word*` searches by prefix; many users will expect partial-word matches.

### 2. Results List

Display each result with:
- A type indicator (icon or label: file, news, msgboard).
- The primary content - file name, article title, or post snippet.
- The source server name.

If you show all types in one list, keep the server's order (grouped by type), or fetch per type. Don't re-sort mixed types by `score`; scores of different types aren't comparable.

Allow the user to select a result for more detail, or connect directly.

### 3. Detail / Preview

Show the full result data:
- **Msgboard post:** nick, body, timestamp, server name + address.
- **News article:** title, poster, date, body, category path, server name + address. Fetch the full body from [Browse News](#browse-news) if needed.
- **File:** name, size, type, path, comment, server name + address.

Provide a "Connect" action that initiates a Hotline connection to the `server_address`.

---

## Implementation Notes

**URL construction:** Always use your language's URL builder (e.g. `URL` class, `url.parse`, `urllib`) to construct request URLs. Never concatenate query parameters as raw strings - the search query and paths may contain special characters (`&`, `#`, `+`, spaces, non-ASCII).

**Timeouts:** Use a reasonable HTTP timeout (10 seconds is a good default). Mnemosyne is a lightweight service that should respond quickly; a hanging request likely indicates a network issue.

**Pagination:** For result sets larger than the default `limit`, use the `offset` parameter. Increment `offset` by `limit` for each subsequent page. Stop when `offset >= total` (search, browse files) or `offset >= total_articles` (browse news). When search reports `total_capped`, don't promise an exact count.

**Caching:** Servers push changes to Mnemosyne as they happen and run a full reconciliation periodically (Janus: every 15 minutes), so the index lags the live server by seconds to minutes. Caching results for a few minutes is safe and reduces load.

**Graceful degradation:** If the health check fails or the URL is not configured, disable search features silently. Do not block the client's core functionality.

**Body truncation:** The search and server detail endpoints truncate `body` fields to the instance's snippet length (200 bytes by default), cut at a character boundary and followed by `...`. The browse news endpoint (`/servers/{id}/news`) returns the full article body as indexed. Servers may themselves send shortened bodies, and Mnemosyne stores at most 64 KiB of any body by default.

**Unknown fields:** Ignore fields you don't recognize; new ones may be added.
