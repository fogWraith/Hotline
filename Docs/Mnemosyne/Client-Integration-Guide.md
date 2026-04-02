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

Mnemosyne is an optional indexing service for the Hotline ecosystem. Hotline servers that opt in periodically sync their content (message board posts, news articles, and file listings) to a Mnemosyne instance, which provides a full-text search API over the aggregated index.  
Do note that the content can be a constellation of different configurations, such as file listings only, or file listings and news articles, or only news articles etc.

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

Your client communicates only with the Mnemosyne HTTP API. The sync between Hotline servers and Mnemosyne happens independently via server-side plugins and is not visible to clients.

---

## Configuration

Your client needs a single user-configurable setting:

| Setting | Description | Example |
|---------|-------------|---------|
| Mnemosyne URL | Base URL of the Mnemosyne instance | `http://mnemosyne.example.com:8980` |

Store this in your client's preferences/settings. There is no registration or account creation required. If the URL is empty or unset, disable search functionality gracefully.

---

## Authentication

All read endpoints are **public** — no API key is required.

If you do provide a key (for future use or internal tooling), it can be passed via:

- **Header:** `X-API-Key: <key>`
- **Query parameter:** `?api_key=<key>`

Invalid keys are rejected with `401`. Omitting the key entirely is always accepted.

---

## Rate Limits

Mnemosyne enforces per-IP rate limiting. The default is **120 requests per minute per IP**. If exceeded, the server returns `429 Too Many Requests`.

Clients should:
- Debounce search input (avoid firing a request on every keystroke).
- Cache results locally when practical.
- Respect `429` responses and back off before retrying.

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
| `400` | Bad request (missing or invalid parameters) |
| `401` | Invalid API key (only if one was provided) |
| `404` | Resource not found (e.g. unknown `server_id`) |
| `429` | Rate limit exceeded |
| `500` | Internal server error |
| `503` | Service degraded (database issue) |

---

## API Reference

All endpoints are prefixed with `/api/v1`. Responses are `application/json`.

---

### Health Check

```
GET /api/v1/health
```

Use this to verify the Mnemosyne instance is reachable before enabling search features.

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
| `q` | string | **yes** | — | Search query text |
| `type` | string | no | all | Filter by content type: `msgboard`, `news`, or `files` |
| `server` | string | no | all | Filter by `server_id` (UUID) |
| `limit` | int | no | `20` | Maximum results to return (hard cap: `100`) |
| `offset` | int | no | `0` | Pagination offset |

**Response:**
```json
{
  "total": 57,
  "results": [
    {
      "type": "msgboard",
      "server_id": "a1b2c3d4-...",
      "server_name": "My Server",
      "server_address": "example.com:5500",
      "score": 12.5,
      "data": {
        "post_id": 42,
        "nick": "admin",
        "body": "This is a truncated snippet of the post...",
        "timestamp": "2026-03-30T10:00:00"
      }
    },
    {
      "type": "news",
      "server_id": "a1b2c3d4-...",
      "server_name": "My Server",
      "server_address": "example.com:5500",
      "score": 8.2,
      "data": {
        "path": "/News/General",
        "article_id": 5,
        "title": "Article Title",
        "poster": "admin",
        "body": "This is a truncated snippet of the article...",
        "date": "2026-03-28"
      }
    },
    {
      "type": "file",
      "server_id": "a1b2c3d4-...",
      "server_name": "My Server",
      "server_address": "example.com:5500",
      "score": 5.0,
      "data": {
        "path": "/Files/Images/",
        "name": "photo.jpg",
        "size": 204800,
        "type": "JPEG",
        "comment": "A nice photo"
      }
    }
  ]
}
```

**Result fields:**

| Field | Type | Description |
|-------|------|-------------|
| `total` | int | Total matching results (may exceed `limit`) |
| `type` | string | Content type: `"msgboard"`, `"news"`, or `"file"` |
| `server_id` | string | UUID of the source server |
| `server_name` | string | Display name of the source server |
| `server_address` | string | Hotline address of the source server (`host:port`) |
| `score` | float | Relevance score (higher = more relevant) |
| `data` | object | Type-specific fields (see below) |

**Data fields by type:**

Message board (`msgboard`):

| Field | Type | Description |
|-------|------|-------------|
| `post_id` | int | Post identifier |
| `nick` | string | Author nickname |
| `body` | string | Post body (truncated to ~200 chars) |
| `timestamp` | string | Post timestamp |

News article (`news`):

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | News category path (e.g. `"/News/General"`) |
| `article_id` | int | Article identifier |
| `title` | string | Article title |
| `poster` | string | Author name |
| `body` | string | Article body (truncated to ~200 chars) |
| `date` | string | Article date |

File (`file`):

| Field | Type | Description |
|-------|------|-------------|
| `path` | string | Directory path containing the file |
| `name` | string | File name |
| `size` | int | File size in bytes |
| `type` | string | Hotline type code (e.g. `"JPEG"`, `"TEXT"`) |
| `comment` | string | File comment (if any) |

---

### List Servers

```
GET /api/v1/servers
```

Lists all servers currently registered with the Mnemosyne index (status `active` or `stale`).

**Query Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `address` | string | no | Filter by exact server address |

**Response:**
```json
{
  "servers": [
    {
      "server_id": "a1b2c3d4-...",
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

Sorted by `last_sync` descending. The `last_sync` field is omitted if the server has never synced.

---

### Server Detail

```
GET /api/v1/servers/{server_id}
```

Detailed view of a single server, including recent content samples. Useful for showing a server preview before the user connects.

**Response:**
```json
{
  "server_id": "a1b2c3d4-...",
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
    {"id": 10, "nick": "admin", "body": "Latest post preview...", "timestamp": "..."}
  ],
  "recent_articles": [
    {"id": 5, "path": "/News", "title": "Latest Update", "poster": "admin", "date": "..."}
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"active"` or `"stale"` |
| `recent_posts` | array | Up to 5 most recent message board posts (body truncated) |
| `recent_articles` | array | Up to 5 most recent news articles (title/poster only, no body) |

Returns `404` if the `server_id` is not found.

---

### Browse News

```
GET /api/v1/servers/{server_id}/news?path=/
```

Browse the news category hierarchy for a specific server. Provides full article bodies (not truncated).

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `path` | string | no | `"/"` | News category path to browse |

**Response:**
```json
{
  "path": "/",
  "categories": [
    {"path": "/General", "name": "General"},
    {"path": "/Announcements", "name": "Announcements"}
  ],
  "articles": [
    {
      "article_id": 1,
      "title": "Welcome",
      "poster": "admin",
      "date": "2026-03-01",
      "body": "Full article body text (not truncated)"
    }
  ]
}
```

- `categories`: direct child categories at the requested path.
- `articles`: articles at the exact path, sorted by `article_id` descending.
- Both arrays are always present (empty `[]` if no items).

---

### Browse Files

```
GET /api/v1/servers/{server_id}/files?path=/
```

Browse the file tree for a specific server (one directory level at a time).

**Query Parameters:**

| Param | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `path` | string | no | `"/"` | Directory path to browse |

**Response:**
```json
{
  "path": "/",
  "files": [
    {
      "path": "/Images/",
      "name": "Images",
      "size": 0,
      "type": "",
      "creator": "",
      "comment": "",
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
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `is_dir` | bool | `true` for directories, `false` for files |
| `type` | string | Hotline 4-char type code (may be empty) |
| `creator` | string | Hotline 4-char creator code (may be empty) |
| `size` | int | File size in bytes (`0` for directories) |
| `comment` | string | File comment (may be empty) |

Sorted directories first, then by name.

---

### Statistics

```
GET /api/v1/stats
```

Aggregate statistics across the entire Mnemosyne index. Useful for an "about" or status display.

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
    {"server_id": "...", "server_name": "Big Server", "total_content": 500}
  ],
  "recently_updated": [
    {"server_id": "...", "server_name": "Fresh Server", "last_sync": "2026-03-31T12:00:00"}
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
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
4. Connect as a guest by default — the client has no way to know what credentials are required.

The address can contain a hostname or IP, with or without a port number. Treat it identically to how your client handles manually-entered server addresses.

---

## Recommended UI Flow

A typical integration provides three views:

### 1. Search Input

- A text field for the search query.
- An optional type filter toggle (all / files / news / msgboard).
- Fire the search request when the user presses Enter (not on every keystroke — see [Rate Limits](#rate-limits)).

### 2. Results List

Display each result with:
- A type indicator (icon or label: file, news, msgboard).
- The primary content — file name, article title, or post snippet.
- The source server name.
- Relevance score (optional).

Allow the user to select a result for more detail, or connect directly.

### 3. Detail / Preview

Show the full result data:
- **Msgboard post:** nick, body, timestamp, server name + address.
- **News article:** title, poster, date, body, category path, server name + address.
- **File:** name, size, type, path, comment, server name + address.

Provide a "Connect" action that initiates a Hotline connection to the `server_address`.

---

## Implementation Notes

**URL construction:** Always use your language's URL builder (e.g. `URL` class, `url.parse`, `urllib`) to construct request URLs. Never concatenate query parameters as raw strings — the search query may contain special characters.

**Timeouts:** Use a reasonable HTTP timeout (10 seconds is a good default). Mnemosyne is a lightweight service that should respond quickly; a hanging request likely indicates a network issue.

**Pagination:** For result sets larger than the default `limit`, use the `offset` parameter. Increment `offset` by `limit` for each subsequent page. Stop when `offset >= total`.

**Caching:** Search results are relatively stable between syncs (every 15 minutes by default). Caching results for short periods is safe and reduces load.

**Graceful degradation:** If the health check fails or the URL is not configured, disable search features silently. Do not block the client's core functionality.

**Body truncation:** The search endpoint truncates `body` fields to approximately 200 characters followed by `...`. The browse news endpoint (`/servers/{id}/news`) returns full article bodies if you need the complete text.
