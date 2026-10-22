# Mnemosyne Indexing – Server Integration Guide

**Specification for pushing a Hotline server's content to the Mnemosyne index, for server developers**

---

## Table of Contents

- [Overview](#overview)
- [What You Need](#what-you-need)
- [Getting a Server Key](#getting-a-server-key)
- [Conventions](#conventions)
  - [Requests](#requests)
  - [Content and Identity](#content-and-identity)
  - [Paths](#paths)
  - [Text and Timestamps](#text-and-timestamps)
  - [Limits](#limits)
- [Server Lifecycle](#server-lifecycle)
- [Sync Strategy](#sync-strategy)
- [API Reference](#api-reference)
  - [Heartbeat](#heartbeat)
  - [Single-Request Sync](#single-request-sync)
  - [Chunked Sync](#chunked-sync)
  - [Deregister](#deregister)
- [Errors and Retries](#errors-and-retries)
- [Rate Limits](#rate-limits)
- [Checklist](#checklist)
- [Reference Implementation](#reference-implementation)

---

## Overview

Mnemosyne is an optional indexing service for the Hotline ecosystem. Servers that opt in push metadata about their **message board**, **threaded news** and **file listing** to a Mnemosyne instance; Hotline clients and websites then search that index to find servers by what they host, before connecting.

```
┌───────────────┐   HTTPS push    ┌───────────┐    HTTP GET    ┌──────────┐
│ Your Hotline  │ ──────────────► │ Mnemosyne │ ◄───────────── │ Clients, │
│ server        │ ◄────────────── │  index    │ ─────────────► │ websites │
└───────────────┘   replies       └───────────┘                └──────────┘
```

Everything is plain HTTP and JSON, so any language with an HTTP client and a JSON encoder will do. Your server only ever **pushes**: Mnemosyne never connects to your Hotline port. You decide what to index: any combination of the three content types, and within each, which items. Leave out anything private (drop boxes, upload folders, members-only areas).

Clients that find your server in the index connect to it like to any other server, at the address you registered.

This guide covers the push side. The search side is in the [Client Integration Guide](Client-Integration-Guide.md).

---

## What You Need

| Setting | Description | Example |
|---------|-------------|---------|
| Mnemosyne URL | Base URL of the instance, no trailing slash | `https://mnemosyne.example.com` |
| Server key | The API key Mnemosyne issued for this server | `msv_3f0c…` |
| Enabled per content type | Which of message board, news and files to index | files + news |

Make the key a secret in your configuration (not world-readable, not logged). It authorizes changing this server's index and nothing else.

---

## Getting a Server Key

Keys belong to an **operator account**: the person or group running one or more Hotline servers. Each registered server gets its own **server key** (prefix `msv_`).

1. **Get an operator account.** Either sign up through the instance's web front end (for example Agora), or call `POST /api/v1/register` with `{"id": "...", "name": "...", "email": "..."}`. The reply contains an operator key (`mop_…`). Some instances require an administrator to approve new accounts before they can register servers.
2. **Register the server** with the operator key:

   ```
   POST /api/v1/operator/servers
   X-API-Key: mop_…
   Content-Type: application/json

   {
     "server_id": "retro-hub",
     "server_name": "Retro Hub",
     "server_address": "hotline.example.com:5500",
     "server_description": "Classic Mac software and news"
   }
   ```

   `server_id` is a permanent identifier you choose: 3–64 characters of `a-z`, `0-9`, `-` and `_`. `server_address` is what clients will connect to: give the public host name and port. The reply contains the **server key**, shown only this once:

   ```json
   {
     "server": {"server_id": "retro-hub", "server_name": "Retro Hub", "status": "pending_first_sync"},
     "api_key": "msv_3f0c…"
   }
   ```

3. **Configure your server** with the Mnemosyne URL and the server key.

An operator can later change the name, address and description (`PATCH /api/v1/operator/servers/{server_id}`), get a new server key (`POST …/{server_id}/rotate-key`, which revokes the old one at once), see sync progress (`GET …/{server_id}/status`), or remove the server and its index (`DELETE …/{server_id}`). Your server software doesn't need any of these; it only needs the server key.

---

## Conventions

### Requests

- All endpoints are under `/api/v1`, take `POST` with a JSON body (`Content-Type: application/json`), and reply with JSON.
- Send the server key in the **`X-API-Key` header**. Never put it in the URL: URLs end up in proxy and server logs, and Mnemosyne refuses server keys there.
- Bodies may be **gzip-compressed** (`Content-Encoding: gzip`). That is worth doing for anything over a few KiB: listings compress well.
- Use HTTPS when the instance offers it; the key travels with every request.
- Send a `User-Agent` naming your server software and version; it helps instance operators diagnose problems.
- Errors come in one envelope: `{"error": {"code": "…", "message": "…"}}`. Branch on the HTTP status and `code`; `message` is for humans.

### Content and Identity

Each content type is a set of items, and every item has an **identity**. Sending an item whose identity already exists **replaces** it; removals name items by identity.

| Type | Item | Identity | Fields |
|------|------|----------|--------|
| Message board | post | `id` | `id` (int), `nick`, `login`, `body`, `timestamp` |
| News | category | `path` | `path`, `name` |
| News | article | `path` + `id` | `id` (int), `path` (its category), `title`, `poster`, `date`, `parent_id` (int, for threading; 0 for none), `body` |
| Files | file or folder | `path` | `path`, `name`, `size` (int, bytes), `type`, `creator`, `comment`, `modified`, `is_dir` (bool) |

- All fields except the identity are optional; missing ones are stored empty (or 0/false).
- Identities must be **stable**: the same post, article or file must keep its identity from one sync to the next. Otherwise every sync looks like everything was deleted and re-added, and clients' links break.
- The three content types are independent: syncing one never touches the others.
- Within one type, news categories and news articles are also independent: removing a category does **not** remove its articles. Remove them explicitly (or send a full news sync).
- `type` and `creator` are the classic Mac four-character codes (`"SITD"`, `"TEXT"`, `"ttxt"`); leave them empty if your server has none.
- **Folders** (`is_dir: true`) are optional. Mnemosyne derives folders from file paths, so a listing of files alone can be browsed folder by folder. Send folder entries only if you have something to say about a folder (a comment). Folders don't count as files and their `size` is ignored.

### Paths

- Paths are **server-relative**: what a Hotline user would see, never your on-disk location. `/Software/Utilities/ResEdit.sit`, not `/home/hotline/files/Software/Utilities/ResEdit.sit`.
- They start with `/` and use `/` as the separator, whatever your platform uses. No trailing slash, not even for folders.
- A file's `path` includes its own name; `name` repeats the last component as it should be displayed.
- News `path` is the category: `/General`, `/General/Mac OS`. An article's `path` is the path of the category it is in.
- A path component may contain any character except `/`. If your server allows `/` in names (classic Mac OS does), replace it, consistently, with something else, e.g. `:`.

### Text and Timestamps

- JSON strings are **UTF-8**. Hotline content is often MacRoman (or another legacy encoding) on disk and on the wire; convert it before sending, or every non-ASCII character will come out wrong in search results.
- Line breaks in bodies: send `\n`. Classic Hotline uses `\r`; convert.
- **Timestamps** (`timestamp`, `date`, `modified`): send RFC 3339 with your offset, e.g. `2026-03-30T12:00:00+02:00` or `2026-03-30T10:00:00Z`. Mnemosyne stores them in UTC. `YYYY-MM-DD HH:MM:SS` without an offset is also accepted and taken as UTC. Anything else is stored as sent, which clients may fail to read, and sorts unpredictably. Leave the field out if you don't know the time.

### Limits

| Limit | Default | When exceeded |
|-------|---------|---------------|
| Request body | 10 MiB (after decompression too) | `413 too_large` |
| `path` | 4096 bytes | `422`; the request is refused, naming the item |
| file `name` | 1024 bytes | `422` |
| `body` (posts, articles) | 64 KiB (per instance) | cut to fit |
| `comment` | 4096 bytes | cut to fit |
| `title`, category `name` | 1024 bytes | cut to fit |
| `nick`, `login`, `poster`, `type`, `creator`, timestamps | 255 bytes | cut to fit |
| `sync_id` | 128 characters | `422` |
| Rows staged by one chunked sync | 10,000,000 (per instance) | `413 too_large` |

Text is cut at a character boundary. If you prefer to decide yourself what is cut (e.g. send only the start of long article bodies), do it before sending.

---

## Server Lifecycle

A registered server moves through these states, which also decide whether its content is public:

```
 register ──► pending_first_sync ──(first full sync)──► active ◄──(heartbeat or sync)──┐
                    ▲                                     │                            │
                    │                          (no contact for 24 h)                   │
                    │                                     ▼                            │
                    └──(no contact for 7 days: ───────── stale ────────────────────────┘
                        content purged)
```

| State | Public? | Meaning |
|-------|---------|---------|
| `pending_first_sync` | no | Mnemosyne holds no content for the server: it is new, deregistered, or its content was purged |
| `active` | yes | Normal operation |
| `stale` | yes | Not heard from (no heartbeat, no sync) for 24 hours; the index may be out of date |

- Only a **full** sync (single-request `mode: "full"`, or a finalized chunked sync) of any content type takes a server out of `pending_first_sync`. Incremental syncs alone never make a server public.
- Any heartbeat or sync makes a `stale` server `active` again.
- After 7 days without contact Mnemosyne **purges the content** and returns the server to `pending_first_sync`. The registration and the server key are kept: when your server comes back, its next heartbeat reply says `full_sync_required: true` and it re-indexes from scratch.
- The 24-hour and 7-day periods are instance defaults; operators can change them.

---

## Sync Strategy

What to send, and when. This is what the reference implementation (Janus) does, and what this guide recommends.

1. **At startup**, after a short delay (so the server is up first), run a **full sync** of each content type you index, then send a heartbeat.
2. **Every 5 minutes**, send a **heartbeat**. If its reply says `full_sync_required: true`, run full syncs of every type you index. Don't rely on startup alone: if your server ran through a long outage of Mnemosyne or of the network, its content may have been purged while your process kept running.
3. **As things change**, send **incremental syncs**: a post written or deleted, an article posted, a file uploaded, renamed, moved or deleted. Batch events over a few seconds rather than sending one request per event.
4. **Every 15 minutes or so**, **reconcile**: compare a cheap signature of each content type (item count; for files also total size) with the one from your last full sync, and run a full sync of any type that changed. This catches changes your event hooks never see: files copied straight into the file folder, edits outside the server, events lost while Mnemosyne was unreachable.
5. **On shutdown**, do nothing special: heartbeats stop, and the index stays as it was until Mnemosyne considers the server stale. Only **deregister** when you stop indexing for good (or want to clear everything).

Choosing between single-request and chunked full syncs:

| Content size | Use |
|--------------|-----|
| Comfortably under 10 MiB of JSON (a few thousand files, posts or articles) | [Single-request sync](#single-request-sync), `mode: "full"` |
| Anything larger, or of unknown size | [Chunked sync](#chunked-sync) |

When in doubt, use chunked sync: it works for any size, and a chunked sync of a small server is one chunk plus the finalize.

**Paging through your content for a chunked sync must visit every item exactly once.** Page by a key that orders all items totally, such as (modification time, path), and resume strictly after the last item sent. Paging by modification time alone and resuming "one second later" skips files that share a second: archives extracted or folders copied in one go put hundreds of files in the same second. (Janus had exactly this bug: a test with 300 files in one second indexed fewer than half.) An item sent twice does no harm (the later copy wins); an item never sent is missing from the index until the next full sync.

---

## API Reference

All sync endpoints require the server key in `X-API-Key`.

---

### Heartbeat

```
POST /api/v1/sync/heartbeat
```

Tells Mnemosyne the server is alive and updates its display name and description.

**Request:**
```json
{
  "server_name": "Retro Hub",
  "description": "Classic Mac software and news"
}
```

| Field | Description |
|-------|-------------|
| `server_name` | Current display name; empty or missing keeps the registered one |
| `description` | Current description; empty or missing keeps the registered one |

**Response:**
```json
{"status": "ok", "full_sync_required": false}
```

| Field | Description |
|-------|-------------|
| `full_sync_required` | `true` when Mnemosyne holds no content for the server (never synced, deregistered, or purged). Run full syncs of every content type you index |

Content counts are computed by Mnemosyne from what you sync; there is nothing to report. (Older senders include `post_count`, `file_count` and `news_count`; they are ignored.)

---

### Single-Request Sync

One request per content type, either a **full** replacement or an **incremental** change.

```
POST /api/v1/sync/msgboard
POST /api/v1/sync/news
POST /api/v1/sync/files
```

**Response** (all modes): `{"status": "ok"}`. The change is applied atomically: all of it, or nothing.

#### Full mode

Replaces everything Mnemosyne holds for that content type with what you send. Items not in the request are removed.

```json
{
  "mode": "full",
  "posts": [
    {"id": 41, "nick": "jo", "login": "jo", "body": "Anyone have ResEdit 2.1.3?", "timestamp": "2026-03-30T12:00:00+02:00"},
    {"id": 42, "nick": "admin", "login": "admin", "body": "In /Software/Utilities.", "timestamp": "2026-03-30T12:05:00+02:00"}
  ]
}
```

```json
{
  "mode": "full",
  "categories": [
    {"path": "/General", "name": "General"},
    {"path": "/General/Mac OS", "name": "Mac OS"}
  ],
  "articles": [
    {"id": 1, "path": "/General", "title": "Welcome", "poster": "admin", "date": "2026-03-01T09:00:00Z", "parent_id": 0, "body": "Welcome to Retro Hub."},
    {"id": 2, "path": "/General", "title": "Re: Welcome", "poster": "jo", "date": "2026-03-01T10:00:00Z", "parent_id": 1, "body": "Thanks!"}
  ]
}
```

```json
{
  "mode": "full",
  "files": [
    {"path": "/Software/Utilities/ResEdit 2.1.3.sit", "name": "ResEdit 2.1.3.sit", "size": 524288,
     "type": "SITD", "creator": "SIT!", "comment": "Apple's resource editor", "modified": "1994-05-02T00:00:00Z"},
    {"path": "/Software", "name": "Software", "is_dir": true, "comment": "Applications and utilities"}
  ]
}
```

A full sync of an empty list is valid and clears that content type.

#### Incremental mode

Applies changes to what Mnemosyne already holds. Within one request, removals are applied before additions, so you can remove and re-add the same identity.

Message board:
```json
{
  "mode": "incremental",
  "added": [{"id": 43, "nick": "jo", "body": "Got it, thanks", "timestamp": "2026-03-30T12:10:00+02:00"}],
  "removed": [41]
}
```

News:
```json
{
  "mode": "incremental",
  "added_categories": [{"path": "/Announcements", "name": "Announcements"}],
  "removed_categories": ["/Old"],
  "added_articles": [{"id": 3, "path": "/Announcements", "title": "Server move", "poster": "admin", "date": "2026-03-31T08:00:00Z", "body": "…"}],
  "removed_articles": [{"path": "/Old", "id": 7}]
}
```

Files:
```json
{
  "mode": "incremental",
  "added": [{"path": "/Uploads/new.sit", "name": "new.sit", "size": 1024, "type": "SITD", "creator": "SIT!"}],
  "removed": ["/Uploads/old.sit"],
  "updated": [{"path": "/Software/Utilities/ResEdit 2.1.3.sit", "comment": "Updated comment"}]
}
```

| Field | Meaning |
|-------|---------|
| `added` / `added_*` | Items to add, or to replace if their identity exists |
| `removed` / `removed_*` | Identities to remove: post IDs, category paths, `{path, id}` for articles, file paths. Unknown ones are ignored |
| `updated` (files only) | Change some fields of an existing file, by `path`: any of `name`, `size`, `comment`. Fields left out keep their value. Unknown paths are ignored |

A **rename or move** is a removal of the old path plus an addition of the new one, in one request. Moving a folder means removing and adding every file under it (or running a full sync of the files).

---

### Chunked Sync

A full sync split over many requests, for content of any size. Mnemosyne stages the chunks and, when you finalize, replaces the content type with what was staged, in one atomic step. Until then, the old content stays searchable.

```
POST /api/v1/sync/msgboard/chunked
POST /api/v1/sync/news/chunked
POST /api/v1/sync/files/chunked
```

**Request:**
```json
{
  "sync_id": "1730000000-482913",
  "chunk_index": 0,
  "finalize": false,
  "files": [
    {"path": "/Software/Utilities/ResEdit 2.1.3.sit", "name": "ResEdit 2.1.3.sit", "size": 524288, "type": "SITD", "creator": "SIT!"}
  ]
}
```

| Field | Description |
|-------|-------------|
| `sync_id` | Identifies this sync run: unique per run, at most 128 characters (a timestamp plus a random number does) |
| `chunk_index` | 0, 1, 2, … in order |
| `finalize` | `true` on the last chunk only |
| `files` / `posts` | Items of this chunk (same fields as in full mode) |
| `categories` and `articles` | For news: either or both, in any chunk |

**Response:**
```json
{"status": "accepted", "sync_id": "1730000000-482913", "chunk_index": 0, "chunks_received": 1}
```

| Field | Description |
|-------|-------------|
| `status` | `"accepted"` for a staged chunk; `"finalized"` once the sync has been applied |
| `chunks_received` | Distinct chunk indexes received for this sync |

**Rules:**

1. Number chunks from 0 without gaps. The finalize chunk must have the **highest** index. It may carry items or be empty; sending all data first and then an empty finalize chunk (`{"sync_id": "…", "chunk_index": N, "finalize": true, "files": []}`) is the simplest.
2. On finalize, Mnemosyne checks that every chunk 0…N arrived. If not, it replies **`409 missing_chunks`** and changes nothing; the staged chunks are kept. Resend the missing ones, then the finalize chunk again. Or simply start over with a new `sync_id`.
3. **Resending a chunk index replaces** what that chunk staged before, so retrying a chunk after a timeout or error is always safe.
4. Starting a sync with a new `sync_id` **abandons any unfinished sync of the same content type**. If your process restarts mid-sync, just start a new one.
5. A sync that receives no chunks for an hour is discarded.
6. An item that appears in several chunks keeps its **last** copy.
7. **Sending the finalize chunk again is safe.** Once a sync is applied, Mnemosyne remembers it for a day. Any chunk of it, including a repeated finalize, is answered `200` with `"status": "finalized"` and changes nothing.
8. Only the **start** of a chunked sync (its first chunk) counts against the full-sync rate limit; a refused start (`429`) has staged nothing. Wait for `Retry-After` and start again.
9. One `sync_id` belongs to one content type; sending it to another type's endpoint is a `409`.

**Sizing and pacing:** about 250 files (or 500 posts) per chunk, and at most 512 KiB of JSON, keeps each request small enough to retry cheaply on a slow uplink. Janus waits 2 seconds between chunks; that is polite, not required.

**Finalize can take time.** Finalizing applies the whole sync at once. Re-syncing unchanged content is quick (about half a second per 25,000 files). A server's **first** sync, or one after its content was purged, writes and indexes everything. Allow minutes for very large servers: give the finalize request a long timeout (Mnemosyne allows 10 minutes; a reverse proxy in front of it may allow less). If the finalize times out on your side anyway, send it again: it is answered `"finalized"` whether the first attempt was applied in the meantime or is applied now.

**Pseudocode** for a chunked file sync:

```
sync_id = now() + "-" + random()
index = 0
cursor = start
loop:
    page = next 250 files after cursor, ordered by (modified, path)
    if page is empty: break
    POST /api/v1/sync/files/chunked {sync_id, chunk_index: index, files: page}
        (retry this same request on network errors, 5xx and 429)
    index += 1
    cursor = last file of page
POST /api/v1/sync/files/chunked {sync_id, chunk_index: index, finalize: true, files: []}
    (long timeout)
```

---

### Deregister

```
POST /api/v1/sync/deregister
```

Removes everything Mnemosyne holds for the server (all three content types and any staged syncs) and returns it to `pending_first_sync`. The registration and key stay; a later full sync brings the server back.

**Request:** `{}`

**Response:** `{"status": "deregistered"}`

Use it when indexing is switched off for good. To remove the registration as well, the operator deletes the server (`DELETE /api/v1/operator/servers/{server_id}`).

---

## Errors and Retries

| Status | Code | Meaning | What to do |
|--------|------|---------|------------|
| `200` | - | Done | |
| `400` | `bad_request` | Invalid JSON, invalid gzip, unknown `mode` | Fix the request; don't retry |
| `401` | `unauthorized` | Key missing, unknown or revoked | Stop syncing and report it: the operator must check the key |
| `403` | `forbidden` | Not a server key, or the operator account is pending approval or disabled | Stop and report it |
| `409` | `missing_chunks` | Finalize before all chunks arrived | Resend the missing chunks, or start over |
| `409` | `conflict` | `sync_id` used for another content type | Start a new sync |
| `413` | `too_large` | Body over 10 MiB, or the sync over the staged-row cap | Send smaller chunks; for the cap, contact the instance operator |
| `415` | `unsupported_encoding` | `Content-Encoding` other than `gzip` | Send gzip or uncompressed |
| `422` | `validation_error` | A field is invalid; the message names the item (e.g. `files[3]: path longer than 4096 bytes`) | Fix or skip that item |
| `429` | `rate_limited` | Rate limit reached | Wait `Retry-After` seconds, then retry |
| `5xx` | `internal_error` | Mnemosyne-side failure | Retry with backoff |

**Every sync request is safe to retry.** Full syncs replace, additions replace by identity, removals of missing items are ignored, and chunks replace their earlier copy. So on a network error, a timeout, `429` or `5xx`, retry the same request with exponential backoff (e.g. 5 s, 10 s, 20 s), a few times. If it still fails, give up for now and let the next reconcile run a full sync: incremental changes that couldn't be sent are caught up by the next full sync.

Never let Mnemosyne problems affect your Hotline server: do all of this in the background, with bounded queues, and drop events rather than block when Mnemosyne is slow or down.

---

## Rate Limits

Per server, with the instance defaults:

| Requests | Limit |
|----------|-------|
| Starting a chunked sync (first chunk of a new `sync_id`) | 4 per hour per content type |
| Heartbeats, single-request syncs, deregister | 1000 per hour |
| Chunks of a sync under way | not limited |

Over a limit: `429` with `Retry-After` (seconds). The recommended [sync strategy](#sync-strategy) stays well within these: a reconcile every 15 minutes starts at most 4 full syncs of a type per hour. Batch incremental events over a few seconds; one request per uploaded file in a busy hour can add up.

---

## Checklist

- [ ] Server key from configuration, sent only in `X-API-Key`, never logged.
- [ ] Paths server-relative, starting with `/`, `/`-separated, no trailing slash; stable identities.
- [ ] Text converted to UTF-8; `\r` converted to `\n`.
- [ ] Timestamps RFC 3339 with offset.
- [ ] Private areas (drop boxes, upload folders) left out.
- [ ] Full sync of each indexed type at startup; heartbeat every 5 minutes; full sync when `full_sync_required` is `true`.
- [ ] Incremental syncs on changes, batched over a few seconds.
- [ ] Periodic reconcile that runs a full sync when a type's signature changed.
- [ ] Chunked sync pages by a total order and resumes strictly after the last item; finalize with the highest index and a long timeout.
- [ ] Retries with backoff on network errors, `429` and `5xx`; no retries on other `4xx`.
- [ ] Sync work in the background; the Hotline server never waits for Mnemosyne.
- [ ] gzip request bodies (optional, recommended).

---

## Reference Implementation

Janus implements this feature. Its configuration (`Mnemosyne:` in `config.yaml`) shows the knobs an implementation typically needs: URL, key, per-type toggles, startup delay, heartbeat and reconcile intervals, chunk size and pacing, retries and timeouts.
