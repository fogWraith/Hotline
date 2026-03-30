# Hotline Tracker Protocol — Version 2

**Authenticated tracker protocol for private Hotline trackers**

---

## Table of Contents

- [Background](#background)
  - [Archaeology](#archaeology)
  - [What Was Found](#what-was-found)
  - [What Was Missing](#what-was-missing)
- [The Design Goals](#design-goals)
- [Backward Compatibility](#backward-compatibility)
- [Protocol Overview](#protocol-overview)
- [Account Model](#account-model)
  - [Account Types](#account-types)
  - [Account Storage](#account-storage)
  - [Permissions](#permissions)
- [TCP — Authenticated Client Listing](#tcp--authenticated-client-listing)
  - [Handshake](#handshake)
  - [Authentication Challenge](#authentication-challenge)
  - [Client Credentials](#client-credentials)
  - [Authentication Result](#authentication-result)
  - [Server List Delivery](#server-list-delivery)
  - [TCP Flow Summary](#tcp-flow-summary)
- [UDP — Authenticated Server Registration](#udp--authenticated-server-registration)
  - [v2 Registration Datagram Format](#v2-registration-datagram-format)
  - [Registration Processing](#registration-processing)
  - [Backward Compatibility with v1 Registration](#backward-compatibility-with-v1-registration)
- [Mixed-Mode Operation](#mixed-mode-operation)
  - [Configuration Modes](#configuration-modes)
  - [Version Negotiation on TCP](#version-negotiation-on-tcp)
  - [Version Detection on UDP](#version-detection-on-udp)
- [Security Considerations](#security-considerations)
- [Implementation Notes](#implementation-notes)
- [Wire Capture Examples](#wire-capture-examples)
- [Relationship to v3](#relationship-to-v3)

---

## Background

### Archaeology

Tracker Protocol v2 was planned by at least two independent development efforts in the Hotline ecosystem, but was never completed or deployed. The evidence comes from two codebases:

**GLoarbLine (C++, Tracker Client/Server):**
- `SMyLogin` struct with 32-byte `psLogin` and 32-byte `psPassword` fields
- `SMyLoginInfo` struct adding `nActive`, `psDateTime[33]`, and reserved bytes — a complete account record
- Full login management UI behind a `NEW_TRACKER` preprocessor flag (set to `0`, disabled)
- Guest/Account radio buttons controlling whether login+password fields are enabled
- The tracker server's TCP handshake code only accepts v1 — v2 was never wired up

**SynHXD (C, Tracker Server):**
- UDP dispatch by version: `0x0001` → `htrk_udp_rcv()`, `0x5801` → `htrkx_udp_rcv()` (empty stub)
- TCP state machine recognizes `TRXL\0\1` (list) and `TRXR\0\1` (register) magic sequences, echoes them back, then does nothing
- `htrkx_send_list()` wrapped in `#if 0`
- `tracker_udp_packet` struct in `trx.h` with a TLV-based field format (version + uniqueid + track_fields)

### What Was Found

| Aspect                    | Source        | Evidence                                              |
|---------------------------|---------------|-------------------------------------------------------|
| Account data structures   | GLoarbLine    | `SMyLogin` (login[32] + password[32])                 |
| Account management UI     | GLoarbLine    | Guest/Account modes, login list CRUD                  |
| UDP v2 version number     | synhxd, shxd  | `0x5801` dispatches to empty handler                  |
| TCP v2 magic sequences    | synhxd, shxd  | `TRXL\0\1` and `TRXR\0\1` recognized but unhandled   |

### What Was Missing

- No actual protocol negotiation between v1 and v2
- No wire format for sending credentials in the TCP handshake
- No wire format for authenticated UDP registration
- No definition of what happens after authentication succeeds or fails

This specification hopefully fills those gaps, using the fragments as a foundation.

---

## The Design Goals

1. **Private trackers** — allow tracker operators to restrict who can browse and who can register
2. **Minimal wire changes** — v2 is a natural extension of v1, not a new protocol
3. **Full backward compatibility** — v1 clients and servers continue to work in mixed mode
4. **Simple account model** — login + password pairs with per-account permissions
5. **No cryptographic overhead** — credentials travel in cleartext like v1 passwords
6. **Faithful to the evidence** — uses Pascal strings, 32-byte field sizes, and the same HTRK magic as found in GLoarbLine's data structures

---

## Backward Compatibility

v2 is a strict superset of v1:

- **Same ports** — TCP/5498 and UDP/5499
- **Same HTRK magic** — the 4-byte magic `"HTRK"` is unchanged
- **Same batch format** — server list response uses identical batch headers and server records
- **Version field distinguishes** — `0x0001` vs `0x0002` in both the TCP handshake and UDP datagram header
- **Mixed-mode operation** — a v2 tracker can serve v1 clients and accept v1 registrations simultaneously, controlled by configuration

A tracker running in v2 mode allowing v1 makes it indistinguishable from a v1 tracker to v1 clients and servers, only when the tracker refuses v1 does it become a fully private tracker.

---

## Protocol Overview

```
v2 TCP Listing (Client → Tracker):

  Client                              Tracker
    │                                    │
    │──── HTRK + version 0x0002 ────────►│
    │                                    │
    │◄──── HTRK + version 0x0002 ────────│  (echo = v2 accepted)
    │◄──── auth_required (1 byte) ───────│  (0x01 = send credentials)
    │                                    │
    │──── login + password ─────────────►│  (Pascal strings)
    │                                    │
    │◄──── auth_result (1 byte) ─────────│  (0x00 = OK, 0x01 = denied)
    │                                    │
    │◄──── server list batches ──────────│  (same as v1)
    │                                    │
    ╳                                    ╳  (connection closed)


v2 UDP Registration (Server → Tracker):

  Server                              Tracker
    │                                    │
    │──── v2 datagram ──────────────────►│  (version 0x0002, with login+password)
    │                                    │
    (fire-and-forget, no response — same as v1)
```

---

## Account Model

### Account Types

v2 defines two roles that can be assigned independently:

| Role       | Grants                                              |
|------------|-----------------------------------------------------|
| `list`     | Permission to fetch the server list via TCP          |
| `register` | Permission to register a server via UDP              |

An account may have one or both roles. A typical private tracker setup:

- **Browsing accounts** (`list` only) — distributed to users who want to browse
- **Server accounts** (`register` only) — distributed to server operators
- **Combined accounts** (`list` + `register`) — distributed to users who actively use Hotline and operate servers of their own

### Account Storage

Each account is a login/password pair with a list of roles, below we have a simplified example accounts file in YAML:

```yaml
# accounts.yaml
# in production, passwords should never be stored in cleartext.
accounts:
  - login: "viewer"
    password: "browsepass"
    roles: [list]

  - login: "myserver"
    password: "regpass"
    roles: [register]

  - login: "combinator"
    password: "combined"
    roles: [list, register]
```

Field constraints (derived from GLoarbLine's `SMyLogin` struct):

| Field    | Max Length | Encoding     | Notes                            |
|----------|-----------|--------------|----------------------------------|
| Login    | 31 bytes  | Pascal string | 1-byte length prefix + data     |
| Password | 31 bytes  | Pascal string | 1-byte length prefix + data     |

The 31-byte limit comes from GLoarbLine's 32-byte buffers where byte 0 is the Pascal length prefix, leaving 31 bytes for content.

### Permissions

When authentication is enabled, the tracker checks the account's roles before processing the request:

| Request Type       | Required Role | Rejection Behavior                     |
|--------------------|---------------|----------------------------------------|
| TCP list request   | `list`        | Send `auth_result = 0x01`, close       |
| UDP registration   | `register`    | Silently discard datagram              |

---

## TCP — Authenticated Client Listing

### Handshake

The v2 handshake begins identically to v1, differing only in the version field.

**Client sends** (6 bytes):

| Offset | Size | Field   | Value    | Description                    |
|--------|------|---------|----------|--------------------------------|
| 0      | 4    | Magic   | `"HTRK"` | `0x4854524B`                   |
| 4      | 2    | Version | `0x0002` | Protocol version 2 (big-endian)|

**Tracker responds** (6 bytes):

| Offset | Size | Field   | Value    | Description                    |
|--------|------|---------|----------|--------------------------------|
| 0      | 4    | Magic   | `"HTRK"` | Echo magic                     |
| 4      | 2    | Version | `0x0002` | Confirm v2 accepted            |

If the tracker does not support v2, it closes the connection. A well-behaved client may retry with v1.

### Authentication Challenge

Immediately after the handshake echo, the tracker sends a 1-byte authentication flag:

| Offset | Size | Field          | Value  | Description                       |
|--------|------|----------------|--------|-----------------------------------|
| 0      | 1    | Auth required  | u8     | `0x01` = credentials required     |
|        |      |                |        | `0x00` = no auth, proceed to list |

If `auth_required` is `0x00`, the tracker skips directly to sending the server list (equivalent to a v1 session for a tracker that supports v2 but has no accounts configured).

### Client Credentials

When `auth_required` is `0x01`, the client sends its credentials as two Pascal strings:

| Offset | Size     | Field          | Type          | Description                   |
|--------|----------|----------------|---------------|-------------------------------|
| 0      | 1        | Login length   | u8            | Length of login (0–31)        |
| 1      | login_len| Login          | bytes         | Login string                  |
| N      | 1        | Password length| u8            | Length of password (0–31)     |
| N+1    | pass_len | Password       | bytes         | Password string               |

**Field sizes:** The login and password are variable-length Pascal strings. The maximum length for each is 31 bytes (matching GLoarbLine's `SMyLogin` 32-byte buffers minus the length prefix).

**Encoding:** Strings are raw bytes. Classic Hotline uses MacRoman; modern implementations may send UTF-8. The tracker performs a byte-exact comparison against stored credentials.

**Timeout:** The tracker should impose a read timeout (10 seconds recommended) for receiving credentials. If the client does not send credentials within the timeout, close the connection.

### Authentication Result

After receiving credentials, the tracker validates them and sends a 1-byte result:

| Offset | Size | Field       | Value  | Description                          |
|--------|------|-------------|--------|--------------------------------------|
| 0      | 1    | Auth result | u8     | `0x00` = authenticated               |
|        |      |             |        | `0x01` = denied (bad login/password) |
|        |      |             |        | `0x02` = denied (insufficient role)  |

On `0x00` (success), the tracker proceeds to send the server list. On any non-zero value, the tracker closes the connection after sending the result byte.

### Server List Delivery

After successful authentication (or when `auth_required` is `0x00`), the server list is sent using the **same batch format as v1**. No changes to batch headers, server records, or batching logic.

### TCP Flow Summary

```
v1 client to v1 tracker:
  C→T: HTRK 0x0001
  T→C: HTRK 0x0001
  T→C: [batches]
  (close)

v2 client to v2 tracker (auth required):
  C→T: HTRK 0x0002
  T→C: HTRK 0x0002
  T→C: 0x01                    ← auth required
  C→T: login + password        ← Pascal strings
  T→C: 0x00                    ← authenticated
  T→C: [batches]
  (close)

v2 client to v2 tracker (no auth):
  C→T: HTRK 0x0002
  T→C: HTRK 0x0002
  T→C: 0x00                    ← no auth needed
  T→C: [batches]
  (close)

v1 client to v2 tracker (v1 allowed):
  C→T: HTRK 0x0001
  T→C: HTRK 0x0001             ← downgrade to v1
  T→C: [batches]
  (close)

v1 client to v2 tracker (v1 denied):
  C→T: HTRK 0x0001
  (close)                       ← tracker rejects v1
```

---

## UDP — Authenticated Server Registration

### v2 Registration Datagram Format

The v2 registration datagram extends the v1 format by appending login and password fields after the existing password field. The header and first three Pascal strings are identical to v1, allowing a v2 datagram to be parsed by a v1 tracker (which would simply ignore the trailing bytes).

**Fixed header** (12 bytes, identical to v1):

| Offset | Size | Field       | Type | Description                                      |
|--------|------|-------------|------|--------------------------------------------------|
| 0      | 2    | Version     | u16  | `0x0002` (big-endian) — identifies v2            |
| 2      | 2    | Port        | u16  | Server's TCP listening port (big-endian)          |
| 4      | 2    | User count  | u16  | Number of connected users (big-endian)            |
| 6      | 2    | Reserved    | u16  | Reserved, set to `0x0000`                         |
| 8      | 4    | PassID      | u32  | Random server instance identifier (big-endian)    |

**Variable data:**

| Offset | Size     | Field            | Type          | Description                          |
|--------|----------|------------------|---------------|--------------------------------------|
| 12     | 1+N      | Name             | Pascal string | Server name (1-byte len + data)      |
| N      | 1+N      | Description      | Pascal string | Server description (1-byte len + data)|
| N      | 1+N      | Password         | Pascal string | v1 tracker password (1-byte len + data)|
| N      | 1+N      | Login            | Pascal string | v2 account login (1-byte len + data) |
| N      | 1+N      | Auth password    | Pascal string | v2 account password (1-byte len + data)|

**Key points:**

- The **version field** is `0x0002` instead of `0x0001`
- The v1 **Password** field is retained for backward compatibility (a v2 server registering on a v1 tracker can still authenticate via the v1 password mechanism)
- The **Login** and **Auth password** fields are appended after the v1 password — this is the v2 extension
- Login and Auth password follow the same 31-byte maximum as TCP credentials
- If a v2 datagram is received by a v1 tracker, the parser stops after the three v1 Pascal strings and ignores the trailing login/password fields (graceful degradation)

### Registration Processing

When a v2 registration datagram arrives:

1. **Parse header** — extract version, port, user count, PassID
2. **Detect version** — if version is `0x0002`, parse the additional login and auth password fields after the v1 password field
3. **Authenticate** — look up the login in the account list. If found and the auth password matches and the account has the `register` role, proceed. Otherwise, silently discard.
4. **Process registration** — v1 duplicate detection, per-IP quota, private IP checks, and ban list all apply as normal after successful authentication

**If the tracker has no accounts configured**, a v2 registration is processed identically to v1 (the login/auth password fields are parsed but ignored).

### Backward Compatibility with v1 Registration

A v2 tracker receiving a v1 registration datagram (version `0x0001`) follows the existing v1 processing path:

- If the tracker is in mixed mode (allowing v1), the v1 registration password is checked against the v1 registration password if configured
- If the tracker requires v2 registration (v1 not allowed), the v1 datagram is silently discarded

---

## Mixed-Mode Operation

### Configuration Modes

The tracker should support three operational modes, configurable independently for listing (TCP) and registration (UDP):

| Mode              | v1 Behavior              | v2 Behavior                     |
|-------------------|--------------------------|----------------------------------|
| `open`            | Accept (like v1 tracker) | Accept (auth checked if accounts exist) |
| `mixed`           | Accept                   | Accept (auth required)           |
| `authenticated`   | Reject                   | Accept (auth required)           |

**`open`** — the tracker acts as a v1 tracker. v1 clients and servers work normally. v2 clients can connect and will be told `auth_required = 0x00`. This is the default for an easy migration path.

**`mixed`** — both v1 and v2 are accepted. v1 clients get the list without authentication. v2 clients must authenticate. This allows a gradual transition where legacy clients still work.

**`authenticated`** — only v2 clients with valid credentials are served. v1 connections are rejected. This is the "private tracker" mode.

Registration and listing modes can be set independently, for instance listening in mixed mode (v1 and v2 clients both accepted) and registration mode in authenticated (only v2 with valid credentials)

This allows configurations like:
- **Public listing, private registration** — anyone can browse, only authorized servers appear
- **Private listing, public registration** — only members can browse, any server can register
- **Fully private** — both listing and registration require credentials

### Version Negotiation on TCP

When a client connects:

1. Read the 6-byte handshake
2. Check the version field:
   - `0x0001` — v1 client
     - If listing mode is `open` or `mixed`: echo HTRK v1, send list (v1 flow)
     - If listing mode is `authenticated`: close connection
   - `0x0002` — v2 client
     - Echo HTRK v2
     - If accounts exist: send `auth_required = 0x01`, await credentials, validate
     - If no accounts: send `auth_required = 0x00`, proceed to list
   - Any other version: close connection

### Version Detection on UDP

When a registration datagram arrives:

1. Read the version field (first 2 bytes):
   - `0x0001` — v1 registration
     - If registration mode is `open` or `mixed`: process as v1
     - If registration mode is `authenticated`: silently discard
   - `0x0002` — v2 registration
     - Parse the extended fields (login + auth password)
     - Validate credentials against account list
     - Process registration if authenticated
   - Any other version: silently discard

---

## Security Considerations

**Credentials are transmitted in cleartext.** This is consistent with v1's cleartext password field and with the Hotline protocol's approach. v2 does not attempt to add cryptographic protection at the protocol level.

**For deployments requiring confidentiality:**
- **v3** provides HMAC-based authentication that avoids sending passwords over the wire

**Brute-force protection:**
- The tracker should implement rate limiting on TCP authentication attempts per IP
- Failed authentication attempts should be logged
- Repeated failures from the same IP may trigger a temporary or permanent ban

**Account storage:**
- Passwords in the accounts file should not be stored in cleartext (bcrypt hashed passwords is recommended)
- The file should be readable only by the tracker process owner (file permissions)

---

## Implementation Notes

### Parser Compatibility

The v2 UDP datagram is designed so that a v1 parser reading it will:

1. Parse the 12-byte header (version field will be `0x0002` — a v1 tracker would reject this)
2. If a parser ignores the version field, it would successfully read name, description, and password (the first three Pascal strings) — the v2 login and auth password fields are simply trailing data it never reaches

This means v2 datagrams are safe to send to unknown trackers — they will either be processed (if the tracker supports v2) or rejected at the version check (if v1-only).

### TCP State Machine

A v2 tracker's TCP connection handler has the following states:

```
AWAIT_HANDSHAKE → (read 6 bytes)
    ├── v1: SEND_LIST_V1 → DONE
    ├── v2: SEND_AUTH_FLAG
    │       ├── no auth needed: SEND_LIST → DONE
    │       └── auth required: AWAIT_CREDENTIALS → (read login+pass)
    │               ├── valid: SEND_AUTH_OK → SEND_LIST → DONE
    │               └── invalid: SEND_AUTH_DENIED → DONE
    └── unknown: CLOSE
```

### Interaction with Existing Features

v2 authentication is checked **before** other access controls:

1. Version check
2. Ban list check
3. **v2 authentication** (if applicable)
4. v1 password check (if v1 registration in mixed mode)
5. Private IP rejection
6. Per-IP quota check
7. Duplicate detection

This ordering ensures that banned IPs are rejected before authentication is attempted (no point authenticating a banned address) and that authentication failures don't consume quota.

---

## Relationship to v3

v2 and v3 serve different purposes and are not competing standards:

| Aspect              | v2                              | v3                                  |
|---------------------|---------------------------------|-------------------------------------|
| **Goal**            | Private trackers with auth      | Modern protocol with rich metadata  |
| **Wire format**     | Extension of v1 Pascal strings  | TLV-based structured fields         |
| **Transport**       | Same UDP/TCP as v1              | TCP + TLS, WebSocket option         |
| **Authentication**  | Cleartext login/password        | HMAC-based, challenge-response      |
| **Server records**  | Same v1 batch format            | Structured metadata with TLV fields |

v2 is the pragmatic, low-effort upgrade for operators who want access control without changing their infrastructure.

A tracker can support v1, v2, and v3 simultaneously on the same ports via version detection.

---

## Wire Capture Examples

### v2 TCP Listing — Authenticated Session

A client connects to a v2 tracker that requires authentication. The client authenticates with login `"viewer"` (6 bytes), password `"letmein"` (7 bytes), and receives a list containing one server.

```
Client → Tracker (6 bytes):
  48 54 52 4B 00 02                     HTRK + version 2

Tracker → Client (6 bytes):
  48 54 52 4B 00 02                     HTRK + version 2 (accepted)

Tracker → Client (1 byte):
  01                                    auth_required = 0x01

Client → Tracker (15 bytes):
  06                                    login length = 6
  76 69 65 77 65 72                     "viewer"
  07                                    password length = 7
  6C 65 74 6D 65 69 6E                  "letmein"

Tracker → Client (1 byte):
  00                                    auth_result = 0x00 (OK)

Tracker → Client (8 + 29 bytes):
  00 01                                 Message type = 1
  00 21                                 Data size = 33
  00 01                                 Total servers = 1
  00 01                                 Batch count = 1
  C0 A8 01 64                           IP = 192.168.1.100
  15 7C                                 Port = 5500
  00 03                                 Users = 3
  00 00                                 Reserved
  09                                    Name length = 9
  4D 79 20 53 65 72 76 65 72            "My Server"
  08                                    Desc length = 8
  57 65 6C 63 6F 6D 65 21              "Welcome!"

(connection closed)
```

### v2 TCP Listing — Authentication Denied

```
Client → Tracker (6 bytes):
  48 54 52 4B 00 02                     HTRK + version 2

Tracker → Client (6 bytes):
  48 54 52 4B 00 02                     HTRK + version 2

Tracker → Client (1 byte):
  01                                    auth_required = 0x01

Client → Tracker (14 bytes):
  05                                    login length = 5
  61 64 6D 69 6E                        "admin"
  06                                    password length = 6
  77 72 6F 6E 67 21                     "wrong!"

Tracker → Client (1 byte):
  01                                    auth_result = 0x01 (denied)

(connection closed)
```

### v2 UDP Registration — Authenticated

A server named `"Cool Server"` (11 bytes) with description `"Join us"` (7 bytes) registers with tracker password `""` (empty), v2 login `"myserver"` (8 bytes), v2 auth password `"s3rv3r"` (6 bytes). Port 5500, 5 users, PassID `0xDEADBEEF`.

```
Server → Tracker (49 bytes):
  00 02                                 Version = 0x0002
  15 7C                                 Port = 5500
  00 05                                 Users = 5
  00 00                                 Reserved
  DE AD BE EF                           PassID
  0B                                    Name length = 11
  43 6F 6F 6C 20 53 65 72 76 65 72      "Cool Server"
  07                                    Desc length = 7
  4A 6F 69 6E 20 75 73                  "Join us"
  00                                    v1 password length = 0 (empty)
  08                                    v2 login length = 8
  6D 79 73 65 72 76 65 72               "myserver"
  06                                    v2 auth password length = 6
  73 33 72 76 33 72                     "s3rv3r"

(no response — fire-and-forget)
```
