# Hotline Tracker HTTP/REST API

**Companion specification to [Tracker Protocol v3](Tracker-Protocol-v3.md)**

---

## Overview

A tracker MAY expose an HTTP/REST API on a configurable port for programmatic access to tracker data and administration. This is an optional convenience layer - the binary protocol on ports 5498/5499 is the canonical interface.

The API is versioned independently of the protocol version at `/api/v1/`.

---

## Authentication

All endpoints except `/api/v1/health` require an API key. Keys are configured in `config.yaml` with a scope of `read` or `admin`.

The key is supplied via the `X-API-Key` header (preferred) or `?api_key=` query parameter.

Scope requirements:
- **read** - access to server listings, tracker info, and stats.
- **admin** - all read access plus ban management, account listing, federation status, and server removal.

---

## Response Format

All responses use JSON with a standard envelope:

```json
{
  "data": { ... },
  "error": ""
}
```

On error, `data` is omitted and `error` contains a human-readable message.

---

## Endpoints

### Health

| Method | Path               | Auth     | Description      |
|--------|--------------------|----------|------------------|
| GET    | `/api/v1/health`   | none     | Liveness check   |

Response:
```json
{"data": {"status": "ok"}}
```

### Read Endpoints (scope: read)

| Method | Path                           | Description                            |
|--------|--------------------------------|----------------------------------------|
| GET    | `/api/v1/servers`              | List all servers                       |
| GET    | `/api/v1/servers/{addr}`       | Get a specific server by ip:port       |
| GET    | `/api/v1/servers/{addr}/info`  | Cached Info Port descriptor for a server |
| GET    | `/api/v1/tracker/info`         | Tracker metadata and capabilities      |
| GET    | `/api/v1/tracker/stats`        | Tracker statistics                     |

#### GET /api/v1/servers

Query parameters:
- `q` - search filter. Case-insensitive substring match against the server
  **name**, **description**, **hostname** and **published address**, the same
  scope as the binary protocol's `SEARCH_TEXT`
  (see [Listing Request](Tracker-Protocol-v3.md#listing-request)). The two share
  one filter, so a query behaves identically over either interface.
- `page` - page number (0-indexed, default 0). Negative values are treated
  as `0`.
- `limit` - results per page (default 100, max 1000). Values below 1 are
  treated as the default.

Response:
```json
{
  "data": {
    "servers": [
      {
        "address": "192.168.1.5",
        "port": 5500,
        "users": 3,
        "name": "My Server",
        "description": "A friendly server",
        "version": 200,
        "hostname": "myserver.example.com",
        "federated": false,
        "last_seen": "2026-03-28T12:00:00Z",
        "metadata": {
          "country_code": "US",
          "server_software": "Janus/3.0",
          "supports_hope": "true"
        }
      }
    ],
    "total": 1,
    "page": 0,
    "limit": 100
  }
}
```

#### GET /api/v1/servers/{addr}

Path parameter `{addr}` uses `ip:port` format (e.g. `10.0.0.1:5500`). IPv6
addresses are bracketed, as `net.JoinHostPort` renders them
(e.g. `[2001:db8::1]:5500`).

Returns a single server object (same structure as list item).

**Not every listed server is addressable by this route.** A record that reached
the tracker through [federation](Tracker-Federation.md) as a hostname-type
address has no IP of its own; it appears in `/api/v1/servers` with `address`
empty and its name in `hostname`, and there is no `{addr}` that names it. Such a
record can only be read from the list endpoint, optionally narrowed with `q`.

This is a consequence of the route being keyed on IP, and is called out because
the list response is otherwise a natural place to get an address to fetch: a
client that assumes every row round-trips into this route will fail on exactly
the federated ones.

#### GET /api/v1/servers/{addr}/info

Returns the [Info Port](../Hotline-Info-Port.md) descriptor the tracker last
obtained for this server, if it has one. `{addr}` follows the same `ip:port`
rules as above.

This is **cached observation, not a live probe**: the tracker refreshes
descriptors on its own schedule, and the endpoint never dials the server on
behalf of the caller. A tracker that is not probing has nothing to return.

Response - the descriptor as published by the server, so the shape is the Info
Port document's, not this one's:

```json
{
  "data": {
    "infoVersion": 1,
    "software": {"name": "Janus", "version": "1.9.9"},
    "server": {"name": "My Server", "description": "A friendly server"},
    "dataPort": 5500,
    "tlsPort": 5510,
    "transport": {
      "hope": {"supported": true},
      "tls": {"supported": true},
      "plaintext": {"supported": true, "accepted": true},
      "compression": {}
    },
    "capabilities": {"largeFiles": true, "voice": true},
    "limits": {"maxClients": 100},
    "users": {"connected": 3}
  }
}
```

Note that each `transport` entry is an object - `supported`, `required`,
`accepted` - not a boolean. A transport can be supported without being accepted
(the operator has it compiled in but switched off) and accepted without being
required, and a client choosing how to connect needs all three. Fields default
to `false` and are omitted when unset, so an empty object means "not offered".

Errors:

| Status | Meaning                                                    |
|--------|------------------------------------------------------------|
| 400    | `{addr}` is not `ip:port`                                   |
| 404    | Info Port probing is disabled on this tracker               |
| 404    | No such server                                              |
| 404    | The server is known but no descriptor has been cached for it |

The three 404 cases are distinguished by the `error` string rather than the
status, since all three mean "nothing to return here".

#### GET /api/v1/tracker/info

```json
{
  "data": {
    "version": "1.0.0",
    "protocol_versions": ["v1", "v2", "v3"],
    "federation_enabled": true,
    "federation_id": "abc123..."
  }
}
```

#### GET /api/v1/tracker/stats

```json
{
  "data": {
    "local_servers": 42,
    "federated_servers": 15,
    "total_servers": 57,
    "bans": 3,
    "uptime_seconds": 86400
  }
}
```

### Admin Endpoints (scope: admin)

| Method | Path                            | Description                      |
|--------|---------------------------------|----------------------------------|
| GET    | `/api/v1/bans`                  | List all ban entries             |
| POST   | `/api/v1/bans`                  | Add a ban entry                  |
| DELETE | `/api/v1/bans/{entry}`          | Remove a ban entry               |
| GET    | `/api/v1/accounts`              | List accounts (no passwords)     |
| DELETE | `/api/v1/servers/{addr}`        | Force-remove a server entry      |
| GET    | `/api/v1/federation/peers`      | Federation peer status           |
| GET    | `/api/v1/federation/identity`   | Federation identity info         |

#### POST /api/v1/bans

Request body:
```json
{"entry": "10.0.0.5"}
```

Accepts IP addresses (`"1.2.3.4"`) or CIDR ranges (`"10.0.0.0/8"`). Changes are persisted to the ban file.

#### DELETE /api/v1/bans/{entry}

Path parameter `{entry}` is the IP or CIDR to remove. Changes are persisted.

#### GET /api/v1/accounts

Returns accounts without passwords:
```json
{
  "data": {
    "accounts": [
      {"login": "admin", "roles": ["admin", "register"]},
      {"login": "viewer", "roles": ["list"]}
    ],
    "total": 2
  }
}
```

#### GET /api/v1/federation/peers

```json
{
  "data": {
    "enabled": true,
    "peers": ["tracker2.example.com:5499", "tracker3.example.com:5499"]
  }
}
```

#### GET /api/v1/federation/identity

```json
{
  "data": {
    "enabled": true,
    "tracker_id": "abc123def456..."
  }
}
```

---

## Configuration

The API is configured in `config.yaml`:

```yaml
api:
  enabled: true
  host: ""        # bind address, empty = all interfaces
  port: 8080
  keys:
    - key: "your-secret-api-key"
      scope: "admin"
    - key: "read-only-key"
      scope: "read"
```

---

## Error Codes

| HTTP Status | Meaning                                     |
|-------------|---------------------------------------------|
| 200         | Success                                     |
| 400         | Bad request (invalid parameters or body)     |
| 401         | Missing API key                              |
| 403         | Invalid key or insufficient scope            |
| 404         | Resource not found                           |
| 503         | No API keys configured (misconfiguration)    |
