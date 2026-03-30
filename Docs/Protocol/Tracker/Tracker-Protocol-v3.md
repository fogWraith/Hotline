# Hotline Tracker Protocol — Version 3

**A modernized tracker protocol for the Hotline Connect ecosystem**

---

## Table of Contents

- [Design Goals](#design-goals)
- [Backward Compatibility](#backward-compatibility)
- [Protocol Overview](#protocol-overview)
- [Transport Layer](#transport-layer)
  - [Port Assignments](#port-assignments)
  - [TLS for TCP](#tls-for-tcp)
  - [DTLS for UDP](#dtls-for-udp)
  - [Dual-Stack Operation](#dual-stack-operation)
- [Session Initialization](#session-initialization)
  - [Handshake](#handshake)
  - [Read Strategy](#read-strategy)
  - [Version Negotiation](#version-negotiation)
  - [Feature Flags](#feature-flags)
  - [Feature Discovery](#feature-discovery)
- [Data Encoding](#data-encoding)
  - [Byte Order](#byte-order)
  - [String Encoding](#string-encoding)
  - [String Fields](#string-fields)
  - [Typed Fields (TLV)](#typed-fields-tlv)
  - [Field Type IDs](#field-type-ids)
- [Server Registration (Server → Tracker)](#server-registration-server--tracker)
  - [Registration Transport](#registration-transport)
  - [Registration Datagram Format](#registration-datagram-format)
  - [Registration Metadata Fields](#registration-metadata-fields)
  - [Registration Acknowledgment](#registration-acknowledgment)
  - [Heartbeat Interval](#heartbeat-interval)
  - [Explicit Deregistration](#explicit-deregistration)
  - [Registration Authentication](#registration-authentication)
- [Server Listing (Client → Tracker)](#server-listing-client--tracker)
  - [Listing Handshake](#listing-handshake)
  - [Listing Request](#listing-request)
  - [Server Record Format](#server-record-format)
  - [Listing Response](#listing-response)
  - [Pagination](#pagination)
  - [Client Authentication](#client-authentication)
- [Structured Metadata](#structured-metadata)
  - [Core Fields](#core-fields)
  - [Address Fields](#address-fields)
  - [Descriptive Fields](#descriptive-fields)
  - [Capability Fields](#capability-fields)
  - [Content Index Fields](#content-index-fields)
  - [Privacy and Visibility Fields](#privacy-and-visibility-fields)
  - [Tracker-Injected Fields](#tracker-injected-fields)
- [Security](#security)
  - [HMAC-Signed Registration](#hmac-signed-registration)
  - [Replay Protection](#replay-protection)
  - [Registration Tokens](#registration-tokens)
  - [Rate Limiting](#rate-limiting)
- [Operations](#operations)
  - [Server Expiry](#server-expiry)
  - [Banning and Access Control](#banning-and-access-control)
  - [Logging](#logging)
- [Migration from v1](#migration-from-v1)
- [Wire Capture Examples](#wire-capture-examples)
- [Quick Reference](#quick-reference)
- [Companion Specifications](#companion-specifications)

---

## Design Goals

1. **Security by default.** TLS on TCP, DTLS or HMAC-signed UDP. No cleartext passwords on the wire.
2. **IPv6 native.** Server records support both IPv4 and IPv6 addresses natively.
3. **Extensible metadata.** Typed key-value (TLV) fields replace fixed struct layouts. New fields can be added without breaking parsers.
4. **Backward compatible.** A v3 tracker can serve v1 and v2 clients and accept v1/v2 server registrations on the same ports.
5. **UTF-8 everywhere.** No encoding ambiguity. All strings are UTF-8.
6. **Tracker authority.** The tracker is the source of truth for what it publishes. Servers provide metadata; the tracker decides what to relay.

---

## Backward Compatibility

A v3 tracker MUST support v1 and v2 protocols on the same ports:

| Scenario                            | Behavior                                                                  |
|-------------------------------------|---------------------------------------------------------------------------|
| v1 client connects (TCP)            | Tracker detects `HTRK\x00\x01`, responds with v1 batch format            |
| v2 client connects (TCP)            | Tracker detects `HTRK\x00\x02`, responds with v2 format and auth challenge if configured |
| v3 client connects (TCP)            | Tracker detects `HTRK\x00\x03`, reads 2 more bytes for feature flags, responds with v3 format |
| v1 server registers (UDP)           | Tracker detects version `0x0001` in header, processes as v1               |
| v2 server registers (UDP)           | Tracker detects version `0x0002` in header, processes as v2 (with optional auth credentials) |
| v3 server registers (UDP)           | Tracker detects version `0x0003` in header, processes v1 header + v3 TLV extension block |
| v1 client → v3-only tracker         | Tracker responds with v1 format (subset of data)                          |
| v3 client → v1-only tracker         | Client receives `HTRK\x00\x01` (6 bytes), falls back to v1 parsing       |
| v3 client → v2-only tracker         | Client receives `HTRK\x00\x02` (6 bytes), falls back to v2 parsing       |

The version field in the handshake magic and the registration header is the discriminator.

### Feature Degradation

When serving a v1 or v2 client, the tracker:
- Strips all TLV metadata from server records.
- Uses the v1/v2 record format exactly.
- Converts IPv6-only servers to omitted entries (v1/v2 cannot represent IPv6 in the server record format).
- Ignores v3 query parameters (v1/v2 clients don't send them).

When a v3 server registers with a v1 tracker:
- The v1 tracker parses the first 12 bytes (version, port, users, reserved, PassID) and the Pascal strings normally.
- The v3 TLV extension block appended after the password string is ignored (the v1 tracker stops reading after the password).
- The version field `0x0003` may cause some v1 trackers to reject the datagram. Servers SHOULD be prepared for this and maintain v1 registrations to v1 trackers if needed.

---

## Protocol Overview

```
┌──────────────┐     UDP/5499 (DTLS)     ┌──────────────────┐    TCP/5498 (TLS)   ┌──────────────┐
│              │ ──────────────────────► │                  │ ◄────────────────── │              │
│  HL Server   │    Registration +       │                  │   Listing Request + │  HL Client   │
│              │    Metadata             │  Tracker Server  │   Search + Auth     │              │
│              │ ◄────────────────────── │                  │ ──────────────────► │              │
│              │    Ack (optional)       │                  │   Server List       │              │
└──────────────┘                         └──────────────────┘                     └──────────────┘
```

Key differences from v1:
- Registration is optionally **acknowledged** — the tracker MAY send a UDP response.
- Listing supports **text search** and **pagination**.
- Server records include **structured metadata** via TLV fields.
- **IPv6** addresses in server records (alongside IPv4 and hostnames).
- All strings are **UTF-8**. All numbers are **big-endian**.
- **HMAC-SHA256** authentication for registration when DTLS is not available.

---

## Transport Layer

### Port Assignments

| Transport     | Port | Purpose                              |
|---------------|------|--------------------------------------|
| TCP (TLS)     | 5498 | Client listing                       |
| UDP (DTLS)    | 5499 | Server registration, acknowledgments |

v3 reuses the same default ports as v1.

### TLS for TCP

TCP connections SHOULD use TLS 1.3 (or TLS 1.2 minimum). The tracker presents a server certificate. Clients SHOULD validate the certificate against trusted CAs for public trackers. Self-signed certificates are acceptable for private trackers when explicitly trusted.

**Fallback:** If TLS negotiation fails, the tracker MAY accept plaintext connections for v1/v2 backward compatibility. A v3-only tracker MAY refuse plaintext entirely.

**Client behavior:** A v3 client SHOULD attempt TLS first on port 5498. If the connection fails, retry without TLS and treat the tracker as v1.

### DTLS for UDP

When both the server and tracker support it, UDP registration SHOULD use DTLS 1.3 (or DTLS 1.2). This encrypts the entire datagram including the password field.

**Negotiation:** The server attempts DTLS on port 5499. If the tracker doesn't respond with a DTLS handshake, the server falls back to plaintext UDP with HMAC-signed datagrams.

### Dual-Stack Operation

The tracker MUST listen on both IPv4 and IPv6 when available. Server records can contain any address type: IPv4, IPv6, or hostname. Clients connecting over IPv4 receive all records regardless of address type. The address type is encoded in the server record format (see [Server Record Format](#server-record-format)). When a server registers with a hostname address, the tracker stores the hostname as-is; clients are responsible for DNS resolution.

---

## Session Initialization

### Handshake

The v3 handshake extends the v1/v2 6-byte magic to 8 bytes by appending 2 bytes of feature flags:

**Client sends** (8 bytes):

| Offset | Size | Field         | Type  | Value     | Description                |
|--------|------|---------------|-------|-----------|----------------------------|
| 0      | 4    | Magic         | bytes | `"HTRK"`  | `0x4854524B`               |
| 4      | 2    | Version       | u16   | `0x0003`  | Protocol version 3         |
| 6      | 2    | Feature flags | u16   | bitmask   | Capabilities offered       |

**Tracker responds** (8 bytes):

| Offset | Size | Field         | Type  | Value     | Description                |
|--------|------|---------------|-------|-----------|----------------------------|
| 0      | 4    | Magic         | bytes | `"HTRK"`  | `0x4854524B`               |
| 4      | 2    | Version       | u16   | `0x0003`  | Confirmed version 3        |
| 6      | 2    | Feature flags | u16   | bitmask   | Capabilities supported     |

The negotiated feature set is the **intersection** (bitwise AND) of the client's and tracker's feature flags.

### Read Strategy

The handshake is variable-length: 6 bytes for v1/v2, 8 bytes for v3. Both sides MUST use the following read strategy:

1. Read **6 bytes** from the connection.
2. Check the version field (bytes 4–5).
3. If version is `0x0003`, read **2 more bytes** for the feature flags.
4. If version is `0x0001` or `0x0002`, the handshake is complete at 6 bytes.

**v1/v2 fallback:** If a v3 client sends 8 bytes to a v1 or v2 tracker, the tracker reads 6 bytes and responds with its version. The extra 2 bytes from the client remain in the TCP receive buffer and are harmless — the tracker immediately begins sending the server list (v1) or auth challenge (v2), and the connection closes after the listing is complete.

### Version Negotiation

| Client sends | Tracker has | Result                                       |
|--------------|-------------|----------------------------------------------|
| v3 (0x0003)  | v3          | v3 session with negotiated features          |
| v3 (0x0003)  | v2 only     | Tracker responds v2; client falls back to v2 |
| v3 (0x0003)  | v1 only     | Tracker responds v1; client falls back to v1 |
| v2 (0x0002)  | v3          | Tracker responds v2; serves v2 format        |
| v1 (0x0001)  | v3          | Tracker responds v1; serves v1 format        |
| v1 (0x0001)  | v1 only     | Normal v1 session                            |

### Feature Flags

Feature flags negotiate capabilities for the TCP listing session. They are transmitted as a 2-byte bitmask in the v3 handshake.

| Bit  | Flag               | Description                                          |
|------|--------------------|------------------------------------------------------|
| 0    | `FEAT_IPV6`        | IPv6 addresses in server records                     |
| 1    | `FEAT_QUERY`       | Client can send search and pagination parameters     |
| 2    | `FEAT_CLIENT_AUTH` | Tracker requires client authentication               |
| 3    | `FEAT_REG_ACK`     | Tracker supports registration acknowledgments (informational) |
| 4    | `FEAT_HMAC`        | Tracker supports HMAC-signed registration (informational) |
| 5–15 | Reserved           | Must be 0                                            |

**Notes:**
- TLV metadata in server records is **mandatory** in v3, not a negotiated flag. If you're speaking v3, TLV is supported.
- `FEAT_REG_ACK` and `FEAT_HMAC` are informational — they communicate what the tracker supports for UDP registration. They do not affect the TCP listing session directly but allow clients to display tracker capabilities.
- Bits 5–15 are reserved for future use. Implementations MUST set them to 0 and MUST ignore unknown flag bits.

### Feature Discovery

A client can discover tracker capabilities without requesting a full listing by connecting, performing the handshake, inspecting the returned feature flags and version, and immediately closing the connection.

---

## Data Encoding

### Byte Order

All multi-byte numeric values are transmitted in **network byte order (big-endian)**, consistent with v1.

### String Encoding

All strings in v3 are **UTF-8**. There is no MacRoman, Shift-JIS, or ISO-8859-1 in v3. Implementations MUST encode and decode all string fields as UTF-8.

When a v3 tracker relays data from a v1 server that registered with MacRoman text, the tracker SHOULD transcode the string to UTF-8 before including it in v3 responses.

### String Fields

v3 uses **2-byte length-prefixed strings** in all v3-specific structures (TLV values, server records):

```
[2 bytes: length (u16, big-endian)] [N bytes: UTF-8 string data]
```

Maximum string length: 65535 bytes. No null terminator.

For v1 backward compatibility in the registration datagram, the v1 section still uses 1-byte Pascal strings. The v3 TLV extension area uses 2-byte lengths within TLV headers.

### Typed Fields (TLV)

v3 metadata uses a Type-Length-Value encoding for extensible fields:

| Offset | Size | Field    | Type  | Description                          |
|--------|------|----------|-------|--------------------------------------|
| 0      | 2    | Field ID | u16   | Identifies the field type            |
| 2      | 2    | Length   | u16   | Length of the value in bytes         |
| 4      | var  | Value    | bytes | Field data (format depends on type)  |

Multiple TLV fields are concatenated. The end of the TLV block is determined by the enclosing structure's field count.

**Value types by field:**

| Value Type | Encoding                                    |
|------------|---------------------------------------------|
| u8         | 1 byte                                      |
| u16        | 2 bytes, big-endian                         |
| u32        | 4 bytes, big-endian                         |
| string     | Raw UTF-8 bytes (length from TLV header)    |
| bytes      | Raw binary (length from TLV header)         |
| bool       | 1 byte: `0x00` = false, `0x01` = true       |
| ipv4       | 4 bytes, network order                      |
| ipv6       | 16 bytes, network order                     |

### Field Type IDs

Fields are grouped by number range:

| Range         | Category                  |
|---------------|---------------------------|
| 0x0001–0x00FF | Core and control fields   |
| 0x0100–0x01FF | Address and network       |
| 0x0200–0x02FF | Descriptive metadata      |
| 0x0300–0x03FF | Capabilities and flags    |
| 0x0400–0x04FF | Content index             |
| 0x0500–0x05FF | Privacy and visibility    |
| 0x0600–0x06FF | Tracker-injected          |
| 0x0800–0x08FF | Security and auth         |
| 0x1000–0x10FF | Query parameters          |
| 0xF000–0xFFFF | Vendor extensions         |

Unknown field IDs MUST be silently ignored by parsers. This ensures forward compatibility when new fields are introduced.

**Reserved ranges** (defined by companion specifications):

| Range         | Companion Spec                                       | Purpose                          |
|---------------|------------------------------------------------------|----------------------------------|
| 0x0400–0x04FF | Content Index                                        | Content statistics and counts    |
| 0x0700–0x07FF | [Federation](tracker-federation.md)                  | Federation identity and control  |

---

## Server Registration (Server → Tracker)

### Registration Transport

v3 registration uses **UDP datagrams** on port 5499, consistent with v1. For security, DTLS SHOULD be used when both sides support it. For v1/v2 backward compatibility, plaintext UDP MUST be accepted.

### Registration Datagram Format

The v3 registration datagram extends the v1 format by appending a TLV extension block after the existing v1 payload:

```
┌────────────────────────────────────────────────────┐
│             v1-Compatible Header (12 bytes)        │
├────────────────────────────────────────────────────┤
│  Version (u16) = 0x0003                            │
│  Port (u16)                                        │
│  User count (u16)                                  │
│  Reserved (u16) = 0x0000                           │
│  PassID (u32)                                      │
├────────────────────────────────────────────────────┤
│             v1-Compatible Strings                  │
├────────────────────────────────────────────────────┤
│  Name length (u8) + Name (Pascal string)           │
│  Description length (u8) + Description (Pascal str)│
│  Password length (u8) + Password (Pascal string)   │
├────────────────────────────────────────────────────┤
│             v3 Extension Block                     │
├────────────────────────────────────────────────────┤
│  Extension magic (u16) = 0x4833  ("H3")            │
│  Extension count (u16)                             │
│  TLV field 1: [ID (u16)] [Len (u16)] [Value...]    │
│  TLV field 2: [ID (u16)] [Len (u16)] [Value...]    │
│  ...                                               │
└────────────────────────────────────────────────────┘
```

**Version field:** Set to `0x0003` to indicate a v3 registration.

**Extension magic:** The 2-byte value `0x4833` (`"H3"`) marks the start of the v3 extension block. This magic MUST only be parsed after confirming the version field is `0x0003` — the version field is the primary v3 discriminator. The extension magic serves as a secondary validation to guard against malformed data.

**Extension count:** The number of TLV fields that follow.

**TLV precedence rule:** When a v3 registration includes both v1 Pascal strings (name, description) and equivalent TLV fields, the **TLV value takes precedence**. The v1 Pascal strings exist solely for backward compatibility with v1 trackers that will never see the extension block. A v3 tracker SHOULD prefer TLV values when present.

A v1 tracker that encounters version `0x0003` will typically reject the datagram (version check fails) or ignore everything after the Pascal strings. A v3 tracker processes both the v1 header/strings and the v3 TLV block.

### Registration Metadata Fields

Servers MAY include any of the following TLV fields in the extension block:

| Field ID | Name               | Type   | Description                                          |
|----------|--------------------|--------|------------------------------------------------------|
| 0x0010   | `DEREGISTER`       | bool   | Deregistration request (see [Explicit Deregistration](#explicit-deregistration)) |
| 0x0100   | `ADDRESS_IPV6`     | ipv6   | Server's IPv6 address (16 bytes)                     |
| 0x0101   | `HOSTNAME`         | string | Server's DNS hostname                                |
| 0x0200   | `SERVER_SOFTWARE`  | string | Software name and version (e.g., `"hlserver/1.5.0"`) |
| 0x0201   | `COUNTRY_CODE`     | string | ISO 3166-1 alpha-2 country code (e.g., `"US"`, `"JP"`) |
| 0x0202   | `REGION`           | string | Freeform region/city (e.g., `"California"`)          |
| 0x0203   | `LANGUAGE`         | string | ISO 639-1 language code (e.g., `"en"`, `"ja"`)      |
| 0x0204   | `MAX_USERS`        | u16    | Maximum user capacity                                |
| 0x0205   | `MATURITY`         | u8     | Content rating: 0=general, 1=mature, 2=adult         |
| 0x0206   | `UPTIME`           | u32    | Server uptime in seconds                             |
| 0x0207   | `RULES_URL`        | string | URL to server rules/terms                            |
| 0x0208   | `BANNER_URL`       | string | URL to server banner image                           |
| 0x0209   | `ICON_URL`         | string | URL to server icon image                             |
| 0x0300   | `PROTOCOL_VERSION` | u16    | Hotline protocol version supported (e.g., `0x0197`)  |
| 0x0301   | `SUPPORTS_HOPE`    | bool   | Server supports HOPE encryption                      |
| 0x0302   | `SUPPORTS_TLS`     | bool   | Server supports TLS connections                      |
| 0x0303   | `TLS_PORT`         | u16    | Dedicated TLS port (if different from base port)     |
| 0x0310   | `TAGS`             | string | Comma-separated tags (e.g., `"chat,files,retro"`)   |
| 0x0450   | `NEWS_COUNT`       | u32    | Number of news articles on the server                |
| 0x0451   | `MSGBOARD_COUNT`   | u32    | Number of message board posts                        |
| 0x0452   | `FILES_COUNT`      | u32    | Total number of files hosted                         |
| 0x0453   | `TOTAL_FILE_SIZE`  | u32    | Total file storage size in bytes                     |
| 0x0500   | `PRIVATE_LISTING`  | bool   | Server is unlisted (findable by direct address only) |
| 0x0800   | `REG_TOKEN`        | bytes  | Registration token (from previous ack, if available) |
| 0x0801   | `HMAC_SHA256`      | bytes  | HMAC-SHA256 signature over the datagram              |
| 0x0802   | `NONCE`            | bytes  | 8-byte nonce for replay protection                   |

All fields are optional. A minimal v3 registration contains only the v1-compatible header and strings (with version set to `0x0003`), plus the extension magic and a count of 0.

### Registration Acknowledgment

When a v3 tracker receives a valid v3 registration, it MAY send a UDP response back to the source address. Acknowledgments are **optional** — the protocol MUST function correctly if the server never receives one.

| Offset | Size | Field              | Type  | Description                                  |
|--------|------|--------------------|-------|----------------------------------------------|
| 0      | 2    | Magic              | u16   | `0x4833` ("H3")                              |
| 2      | 1    | Status             | u8    | Result code                                  |
| 3      | 2    | Heartbeat interval | u16   | Recommended interval in seconds (big-endian) |
| 5      | 2    | Extension count    | u16   | Number of TLV fields following               |
| 7      | var  | TLV fields         | TLV[] | Optional fields (token, error message, etc.) |

**Status codes:**

| Code | Name          | Description                                     |
|------|---------------|-------------------------------------------------|
| 0x00 | `ACK_OK`      | Registration accepted                           |
| 0x01 | `ACK_DENIED`  | Password incorrect or authentication failed     |
| 0x02 | `ACK_BANNED`  | Source IP is banned                              |
| 0x03 | `ACK_QUOTA`   | Per-IP server quota exceeded                    |
| 0x04 | `ACK_FULL`    | Tracker has reached maximum server capacity     |
| 0x05 | `ACK_INVALID` | Malformed datagram                              |
| 0xFF | `ACK_ERROR`   | Generic error                                   |

**TLV fields in acknowledgment:**

| Field ID | Name           | Type   | Description                                        |
|----------|----------------|--------|----------------------------------------------------|
| 0x0800   | `REG_TOKEN`    | bytes  | Token the server SHOULD include in future heartbeats |
| 0x0810   | `ERROR_MSG`    | string | Human-readable error description                   |
| 0x0811   | `TRACKER_NAME` | string | Tracker's display name                             |

**Design principle:** Nothing in the protocol depends on the server receiving an acknowledgment:
- If the server receives a `REG_TOKEN`, it SHOULD include it in subsequent heartbeats for anti-hijacking protection.
- If no `REG_TOKEN` is received (ack lost, v1 tracker, or acks disabled), heartbeats work normally — the tracker identifies the server by IP:port.
- The heartbeat interval defaults to 300 seconds if no ack is received.

**v1/v2 servers** do not expect acknowledgments and will ignore any UDP response. The tracker MUST NOT send acknowledgments for v1 or v2 registrations.

### Heartbeat Interval

The tracker communicates its preferred heartbeat interval via the acknowledgment:

- **Default:** 300 seconds (5 minutes), consistent with v1 behavior.
- The tracker MAY adjust this based on load (e.g., 600 seconds under heavy load).
- The server SHOULD respect the tracker's requested interval when received.
- If no acknowledgment is received, the server uses its configured default (typically 300 seconds).

### Explicit Deregistration

A server can request immediate removal from the listing by sending a v3 registration datagram containing the `DEREGISTER` TLV field:

| Field ID | Name         | Type | Value  | Description            |
|----------|--------------|------|--------|------------------------|
| 0x0010   | `DEREGISTER` | bool | `0x01` | Deregistration request |

The deregistration datagram uses the standard v3 format: v1-compatible header, Pascal strings (may be empty), H3 extension magic, and TLV fields including `DEREGISTER`. The `Port` and `PassID` fields in the v1 header MUST match the registered entry.

If the server has a `REG_TOKEN`, it SHOULD include it in the deregistration datagram to authenticate the request.

The tracker removes the entry immediately and MAY send an acknowledgment with `ACK_OK`.

### Registration Authentication

v3 supports HMAC-based authentication alongside the legacy v1 plaintext password:

1. **Shared secret:** The server and tracker share a pre-configured password/key.
2. **HMAC computation:** The server computes HMAC-SHA256 over the entire datagram, with the `HMAC_SHA256` TLV value replaced by 32 zero bytes as a placeholder during computation.
3. **Nonce:** The server includes an 8-byte random nonce (`NONCE` TLV) in every registration. The tracker rejects datagrams with recently-seen nonces (replay protection window of 10 minutes).
4. **Verification:** The tracker extracts the HMAC, zeros the field, recomputes HMAC-SHA256 over the modified datagram, and compares using **constant-time comparison**. Mismatch → reject with `ACK_DENIED`.

**Fallback:** The v1 password field is still present in the datagram. If a server includes both a v1 password and an HMAC, the v3 tracker uses the HMAC and ignores the plaintext password. If only the v1 password is present (no HMAC TLV), the tracker falls back to plaintext comparison.

---

## Server Listing (Client → Tracker)

### Listing Handshake

After the v3 handshake (6 bytes + 2 more after version check), the client sends a **listing request**. In v1, the tracker immediately sends the server list after the handshake — no explicit request is needed. In v3, the client sends a request that may include search parameters.

### Listing Request

The client sends:

| Offset | Size | Field        | Type  | Description                                      |
|--------|------|--------------|-------|--------------------------------------------------|
| 0      | 2    | Request type | u16   | `0x0001` = list servers                          |
| 2      | 2    | Field count  | u16   | Number of TLV fields following (may be 0)        |
| 4      | var  | TLV fields   | TLV[] | Search and pagination parameters (optional)      |

**Query parameters (TLV fields in the request):**

| Field ID | Name          | Type   | Description                                      |
|----------|---------------|--------|--------------------------------------------------|
| 0x1001   | `SEARCH_TEXT` | string | Case-insensitive substring match against server name and description |
| 0x1010   | `PAGE_OFFSET` | u16    | Pagination: skip this many records               |
| 0x1011   | `PAGE_LIMIT`  | u16    | Pagination: return at most this many records     |

A request with `Field count = 0` is equivalent to a v1 listing request — the tracker returns all servers.

When `SEARCH_TEXT` is provided, the tracker returns only servers whose name or description contains the search string (case-insensitive substring match). Client-side filtering is adequate for more complex queries at typical tracker scales (fewer than a few hundred servers).

### Server Record Format

v3 server records use a fixed header followed by optional TLV fields:

| Offset | Size | Field           | Type   | Description                                          |
|--------|------|-----------------|--------|------------------------------------------------------|
| 0      | 1    | Address type    | u8     | `0x04` = IPv4, `0x06` = IPv6, `0x48` = hostname     |
| 1      | var  | Address         | bytes  | 4 bytes (IPv4), 16 bytes (IPv6), or 2+N bytes (hostname) |
| N      | 2    | Port            | u16    | Server's TCP listening port                          |
| N+2    | 2    | User count      | u16    | Connected user count                                 |
| N+4    | 2    | Name length     | u16    | Length of server name in bytes                       |
| N+6    | var  | Name            | bytes  | Server name (UTF-8)                                  |
| M      | 2    | Description len | u16    | Length of description in bytes                       |
| M+2    | var  | Description     | bytes  | Server description (UTF-8)                           |
| P      | 2    | TLV count       | u16    | Number of TLV metadata fields                        |
| P+2    | var  | TLV fields      | TLV[]  | Metadata (tags, geo, capabilities, etc.)             |

**Address type values:**

| Value  | Name     | Address size | Description                               |
|--------|----------|-------------|-------------------------------------------|
| `0x04` | IPv4     | 4 bytes     | IPv4 address (e.g., `192.168.1.1`)        |
| `0x06` | IPv6     | 16 bytes    | IPv6 address (e.g., `2001:db8::1`)        |
| `0x48` | Hostname | 2+N bytes   | u16 length prefix + UTF-8 hostname string |

For hostname addresses, the address field is encoded as a 2-byte big-endian length prefix followed by the UTF-8 hostname bytes. The client is responsible for DNS resolution.

**Key differences from v1:**
- Address type byte allows IPv4, IPv6, and hostname addresses in the same listing.
- Name and description use 2-byte length prefixes (up to 65535 bytes).
- TLV metadata block appended per record.
- The v1 "Flags/Reserved" field is removed — its purpose is served by TLV metadata.

### Listing Response

The tracker responds with:

| Offset | Size | Field          | Type     | Description                                |
|--------|------|----------------|----------|--------------------------------------------|
| 0      | 2    | Response type  | u16      | `0x0001` = server list                     |
| 2      | 4    | Total size     | u32      | Total response payload size in bytes       |
| 6      | 2    | Total servers  | u16      | Total matching server count                |
| 8      | 2    | Record count   | u16      | Number of server records in this message   |
| 10     | var  | Server records | record[] | Serialized server records                  |

**Key differences from v1:**
- `Total size` is u32 (4 bytes) instead of u16 — supports larger response payloads.
- The response is a single logical message. TCP framing handles arbitrary sizes.

### Pagination

When `PAGE_OFFSET` and `PAGE_LIMIT` are provided in the request:

- `Total servers` reflects the **full** matching count (not just the page).
- `Record count` reflects the number of records in this page.
- The client can request subsequent pages by incrementing `PAGE_OFFSET`.

If pagination parameters are not provided, the tracker returns all matching servers in a single response.

### Client Authentication

When the tracker sets `FEAT_CLIENT_AUTH` in the handshake, the client must authenticate after the handshake and before sending a listing request:

**Auth request (client → tracker):**

| Offset | Size | Field       | Type   | Description                   |
|--------|------|-------------|--------|-------------------------------|
| 0      | 2    | Auth type   | u16    | `0x0001` = password           |
| 2      | 2    | Field count | u16    | Number of TLV auth fields     |
| 4      | var  | TLV fields  | TLV[]  | Login/password fields         |

| Field ID | Name         | Type   | Description          |
|----------|-------------|--------|----------------------|
| 0x0820   | `AUTH_LOGIN` | string | Username             |
| 0x0821   | `AUTH_PASS`  | string | Password             |

**Auth response (tracker → client):**

| Offset | Size | Field       | Type  | Description                          |
|--------|------|-------------|-------|--------------------------------------|
| 0      | 1    | Status      | u8    | 0x00 = success, 0x01 = denied       |
| 1      | 2    | Field count | u16   | Optional TLV fields                  |
| 3      | var  | TLV fields  | TLV[] | Error message, etc.                  |

On success, the client proceeds with the listing request. On failure, the tracker closes the connection.

---

## Structured Metadata

The following TLV fields may appear in server records in listing responses. They are populated from registration metadata and from tracker-injected data.

### Core Fields

These fields are always present in the server record fixed header (not TLV):

| Field            | Source       | Notes                                    |
|------------------|--------------|------------------------------------------|
| Address (IP)     | Fixed header | IPv4, IPv6, or hostname                  |
| Port             | Fixed header | TCP listening port                       |
| User count       | Fixed header | Current connected users                  |
| Name             | Fixed header | Server display name (UTF-8)              |
| Description      | Fixed header | Server description (UTF-8)               |

### Address Fields

| Field ID | Name           | Type | Description                               |
|----------|----------------|------|-------------------------------------------|
| 0x0100   | `ADDRESS_IPV6` | ipv6 | IPv6 address (if server is dual-stack)    |

### Descriptive Fields

| Field ID | Name              | Type   | Description                                  |
|----------|-------------------|--------|----------------------------------------------|
| 0x0200   | `SERVER_SOFTWARE` | string | Software identifier (e.g., `"hlserver/1.5.0"`) |
| 0x0201   | `COUNTRY_CODE`    | string | ISO 3166-1 alpha-2 (server-declared)         |
| 0x0202   | `REGION`          | string | Freeform region or city                      |
| 0x0203   | `LANGUAGE`        | string | ISO 639-1 language code                      |
| 0x0204   | `MAX_USERS`       | u16    | Maximum user capacity                        |
| 0x0205   | `MATURITY`        | u8     | 0=general, 1=mature, 2=adult                 |
| 0x0206   | `UPTIME`          | u32    | Server process uptime in seconds             |
| 0x0207   | `RULES_URL`       | string | URL to server rules/terms of service         |
| 0x0208   | `BANNER_URL`      | string | URL to server banner image                   |
| 0x0209   | `ICON_URL`        | string | URL to server icon                           |
| 0x0310   | `TAGS`            | string | Comma-separated tags                         |

### Capability Fields

| Field ID | Name               | Type | Description                               |
|----------|--------------------|------|-------------------------------------------|
| 0x0300   | `PROTOCOL_VERSION` | u16  | Hotline protocol version supported        |
| 0x0301   | `SUPPORTS_HOPE`    | bool | Server supports HOPE encryption           |
| 0x0302   | `SUPPORTS_TLS`     | bool | Server supports TLS connections           |
| 0x0303   | `TLS_PORT`         | u16  | Dedicated TLS port if different           |

### Content Index Fields

These fields convey content statistics about the server. They may be included by the server in its registration, or injected by the tracker when a content indexing service (such as Mnemosyne) is available.

| Field ID | Name              | Type | Description                                |
|----------|-------------------|------|--------------------------------------------|
| 0x0450   | `NEWS_COUNT`      | u32  | Number of news articles                    |
| 0x0451   | `MSGBOARD_COUNT`  | u32  | Number of message board posts              |
| 0x0452   | `FILES_COUNT`     | u32  | Total number of hosted files               |
| 0x0453   | `TOTAL_FILE_SIZE` | u32  | Total file storage size in bytes           |

When injected by the tracker, these fields are populated from an external content indexing service and cached with a configurable TTL. When provided by the server in a v3 registration, the server-declared values are used unless the tracker has a more authoritative source.

### Privacy and Visibility Fields

| Field ID | Name              | Type | Description                                |
|----------|-------------------|------|--------------------------------------------|
| 0x0500   | `PRIVATE_LISTING` | bool | Server is unlisted in public results       |

Private servers are omitted from listing responses. They can be accessed by clients who know the server's address and port directly.

### Tracker-Injected Fields

These fields are set by the tracker, not by the registering server. They MUST NOT appear in registration datagrams — the tracker ignores them if received.

| Field ID | Name               | Type | Description                                |
|----------|--------------------|------|--------------------------------------------|
| 0x0600   | `IS_PROMOTED`      | bool | Tracker-pinned entry (manually added by operator) |
| 0x0601   | `FIRST_SEEN`       | u32  | Unix timestamp of first registration       |
| 0x0602   | `LAST_HEARTBEAT`   | u32  | Unix timestamp of last successful heartbeat |

**`IS_PROMOTED`** marks entries that the tracker operator has manually added or pinned (equivalent to v1 "fake servers" and custom list entries). Promoted entries:
- Are displayed in listings with a flag so clients can distinguish them.
- Never expire.

---

## Security

### HMAC-Signed Registration

When DTLS is not available, the server SHOULD sign its registration datagrams using HMAC-SHA256:

1. Generate an 8-byte random nonce.
2. Include the nonce as `NONCE` TLV (0x0802) in the extension block.
3. Compute HMAC-SHA256 over the complete datagram, with the `HMAC_SHA256` TLV present but its value set to 32 zero bytes as a placeholder.
4. Replace the placeholder with the actual HMAC value.
5. Send the datagram.

The tracker verifies by:

1. Extracting the HMAC TLV value, replacing it with 32 zero bytes.
2. Computing HMAC-SHA256 over the modified datagram using the shared secret.
3. Comparing the computed HMAC to the extracted value using **constant-time comparison**.
4. Checking the nonce against a sliding window of recently seen nonces.

### Replay Protection

The `NONCE` field (8 bytes) provides replay protection:

- The server generates a unique random nonce for every datagram.
- The tracker maintains a set of recently seen nonces (keyed by source IP + nonce, with a 10-minute window).
- Duplicate nonces within the window are rejected.

### Registration Tokens

The tracker MAY issue a **registration token** (`REG_TOKEN`) in the acknowledgment. This provides anti-hijacking protection:

1. Server sends initial registration (no token).
2. Tracker responds with `ACK_OK` + `REG_TOKEN` TLV.
3. Server includes the `REG_TOKEN` in subsequent heartbeats.
4. Tracker validates the token before accepting heartbeat updates.

**Graceful degradation:** If the server never receives the ack (packet loss, NAT issues), it continues sending heartbeats without a token. The tracker MUST accept tokenless heartbeats from the original source IP — the token enhances security but is not required for basic operation.

### Rate Limiting

Trackers SHOULD implement rate limiting:

| Target                   | Recommended Limit       | Scope         |
|--------------------------|-------------------------|---------------|
| UDP registrations        | 10 per minute per IP    | Registration  |
| TCP listing connections  | 30 per minute per IP    | Listing       |
| Failed auth attempts     | 5 per minute per IP     | All           |

Exceeded limits result in silent discard (UDP) or connection close (TCP).

---

## Operations

### Server Expiry

v3 uses timestamp-based expiry exclusively:

- Each entry stores `last_heartbeat` (Unix timestamp).
- Cleanup runs every 60 seconds.
- Default expiry threshold: **600 seconds** (10 minutes = 2 missed heartbeats at 300s interval).
- The threshold is configurable per tracker.
- Promoted entries (`IS_PROMOTED`) never expire.

### Banning and Access Control

v3 inherits all v1 banning concepts and adds:

| Feature                    | v1   | v3  | Notes                                         |
|----------------------------|------|-----|-----------------------------------------------|
| IP ban list                | Yes  | Yes | Wildcard patterns, CIDR notation added in v3  |
| Separate reg/list bans     | Some | Yes | Standard in v3                                |
| Per-IP quota               | Some | Yes | Standard in v3                                |
| Private IP rejection       | Some | Yes | Automatic for 0/127/10/172.16-31/192.168      |
| CIDR notation              | No   | Yes | `192.168.0.0/16` in addition to wildcards     |

### Logging

Recommended log events (in addition to v1 recommendations):

- Feature negotiation result (client version, negotiated features)
- HMAC verification success/failure
- Token issuance and validation
- Client authentication success/failure
- Deregistration received

---

## Migration from v1

### For Tracker Implementors

1. Start with a working v1 (and optionally v2) tracker.
2. Add v3 handshake detection: read 6 bytes; if version is `0x0003`, read 2 more for feature flags.
3. Implement TLV parsing for the v3 extension block in registration datagrams (detect H3 magic `0x4833` only after confirming version `0x0003`).
4. Add v3 listing response format with TLV metadata per record.
5. Implement v3 listing request parsing (search text, pagination).
6. Add registration acknowledgment (optional).
7. Layer on optional features: client auth, HMAC, TLS/DTLS.

### For Server Implementors

1. Continue sending v1 registrations to v1 trackers.
2. For v3 trackers: set version to `0x0003`, append the extension magic `0x4833`, and include desired TLV metadata fields.
3. Handle acknowledgment responses (optional — ignore if not needed).
4. Implement HMAC signing when the tracker requires it.
5. Support explicit deregistration via `DEREGISTER` TLV.

### For Client Implementors

1. Read 6 bytes from the tracker. Check the version.
2. If version is `0x0003`, read 2 more bytes for feature flags. Send a listing request.
3. If version is `0x0001` or `0x0002`, fall back to v1/v2 parsing.
4. Parse v3 server records (address type byte, 2-byte string lengths, TLV metadata).
5. Optional: implement search and pagination.

---

## Wire Capture Examples

### v3 TCP Handshake with Feature Flags

A client connects to a v3 tracker. Client offers IPv6 + Query. Tracker supports IPv6 + Query + RegAck + HMAC.

```
Client → Tracker (8 bytes):
  48 54 52 4B                           "HTRK" magic
  00 03                                 Version = 3
  00 03                                 Feature flags = FEAT_IPV6 | FEAT_QUERY

Tracker → Client (8 bytes):
  48 54 52 4B                           "HTRK" magic
  00 03                                 Version = 3
  00 1B                                 Feature flags = FEAT_IPV6 | FEAT_QUERY |
                                                        FEAT_REG_ACK | FEAT_HMAC

Negotiated = 0x0003 & 0x001B = 0x0003 (IPv6 + Query)
```

### v3 Registration Datagram with TLV Extension

A server named `"Retro Hub"` (9 bytes), description `"Welcome"` (7 bytes), no v1 password, at port 5500 with 12 users, PassID `0xCAFEBABE`. The extension block includes `SERVER_SOFTWARE`, `TAGS`, and `SUPPORTS_HOPE`.

```
Server → Tracker (UDP, 76 bytes):

  v1-Compatible Header (12 bytes):
  00 03                                 Version = 0x0003
  15 7C                                 Port = 5500
  00 0C                                 Users = 12
  00 00                                 Reserved
  CA FE BA BE                           PassID

  v1-Compatible Strings:
  09                                    Name length = 9
  52 65 74 72 6F 20 48 75 62            "Retro Hub"
  07                                    Desc length = 7
  57 65 6C 63 6F 6D 65                  "Welcome"
  00                                    Password length = 0 (empty)

  v3 Extension Block:
  48 33                                 H3 magic (0x4833)
  00 03                                 Extension count = 3

  TLV Field 1 — SERVER_SOFTWARE:
  02 00                                 Field ID = 0x0200
  00 0A                                 Length = 10
  4A 61 6E 75 73 2F 33 2E 30 2E        "Janus/3.0."

  TLV Field 2 — TAGS:
  03 10                                 Field ID = 0x0310
  00 0A                                 Length = 10
  63 68 61 74 2C 72 65 74 72 6F        "chat,retro"

  TLV Field 3 — SUPPORTS_HOPE:
  03 01                                 Field ID = 0x0301
  00 01                                 Length = 1
  01                                    Value = true
```

### v3 Registration Acknowledgment

The tracker accepts the registration and responds with a heartbeat interval of 300 seconds, a registration token, and the tracker name.

```
Tracker → Server (UDP, 37 bytes):

  48 33                                 H3 magic (0x4833)
  00                                    Status = ACK_OK
  01 2C                                 Heartbeat interval = 300 seconds
  00 02                                 Extension count = 2

  TLV Field 1 — REG_TOKEN:
  08 00                                 Field ID = 0x0800
  00 10                                 Length = 16
  A1 B2 C3 D4 E5 F6 07 18              Token (16 random bytes)
  29 3A 4B 5C 6D 7E 8F 90

  TLV Field 2 — TRACKER_NAME:
  08 11                                 Field ID = 0x0811
  00 05                                 Length = 5
  41 72 67 75 73                        "Argus"
```

### v3 Explicit Deregistration

A server sends a deregistration request with its previously received token.

```
Server → Tracker (UDP, 42 bytes):

  v1-Compatible Header (12 bytes):
  00 03                                 Version = 0x0003
  15 7C                                 Port = 5500 (must match registered entry)
  00 00                                 Users = 0
  00 00                                 Reserved
  CA FE BA BE                           PassID (must match)

  v1-Compatible Strings:
  00                                    Name = empty
  00                                    Description = empty
  00                                    Password = empty

  v3 Extension Block:
  48 33                                 H3 magic
  00 02                                 Extension count = 2

  TLV Field 1 — DEREGISTER:
  00 10                                 Field ID = 0x0010
  00 01                                 Length = 1
  01                                    Value = true

  TLV Field 2 — REG_TOKEN:
  08 00                                 Field ID = 0x0800
  00 10                                 Length = 16
  A1 B2 C3 D4 E5 F6 07 18              Token received from ack
  29 3A 4B 5C 6D 7E 8F 90
```

### v3 Listing Request with Search

A client requests servers matching `"retro"` with a page limit of 25.

```
Client → Tracker (after handshake):

  00 01                                 Request type = 0x0001 (list servers)
  00 02                                 Field count = 2

  TLV Field 1 — SEARCH_TEXT:
  10 01                                 Field ID = 0x1001
  00 05                                 Length = 5
  72 65 74 72 6F                        "retro"

  TLV Field 2 — PAGE_LIMIT:
  10 11                                 Field ID = 0x1011
  00 02                                 Length = 2
  00 19                                 Value = 25
```

### v3 Listing Response with Server Record

The tracker responds with one matching server. The server record uses IPv4 addressing and includes TLV metadata.

```
Tracker → Client:

  Response Header (10 bytes):
  00 01                                 Response type = 0x0001
  00 00 00 54                           Total size = 84 bytes
  00 01                                 Total servers = 1
  00 01                                 Record count = 1

  Server Record:
  04                                    Address type = IPv4
  C0 A8 01 64                           IP = 192.168.1.100
  15 7C                                 Port = 5500
  00 0C                                 Users = 12
  00 09                                 Name length = 9
  52 65 74 72 6F 20 48 75 62            "Retro Hub"
  00 07                                 Description length = 7
  57 65 6C 63 6F 6D 65                  "Welcome"
  00 03                                 TLV count = 3

  TLV Field 1 — SERVER_SOFTWARE:
  02 00                                 Field ID = 0x0200
  00 0A                                 Length = 10
  4A 61 6E 75 73 2F 33 2E 30 2E        "Janus/3.0."

  TLV Field 2 — TAGS:
  03 10                                 Field ID = 0x0310
  00 0A                                 Length = 10
  63 68 61 74 2C 72 65 74 72 6F        "chat,retro"

  TLV Field 3 — SUPPORTS_HOPE:
  03 01                                 Field ID = 0x0301
  00 01                                 Length = 1
  01                                    true
```

---

## Quick Reference

### Protocol Constants

| Constant            | Value       | Description                                |
|---------------------|-------------|--------------------------------------------|
| `HTRK_MAGIC`       | `0x4854524B` | ASCII `"HTRK"` — handshake magic          |
| `H3_MAGIC`         | `0x4833`    | ASCII `"H3"` — v3 extension block magic    |
| `HTFD_MAGIC`       | `0x48544644` | ASCII `"HTFD"` — federation magic         |
| `TCP_PORT`         | `5498`      | Default TCP listing port                   |
| `UDP_PORT`         | `5499`      | Default UDP registration port              |
| `VERSION_V1`       | `0x0001`    | Protocol version 1                         |
| `VERSION_V2`       | `0x0002`    | Protocol version 2                         |
| `VERSION_V3`       | `0x0003`    | Protocol version 3                         |

### Feature Flags (v3 Handshake)

| Bit | Mask     | Name              | Description                           |
|-----|----------|-------------------|---------------------------------------|
| 0   | `0x0001` | `FEAT_IPV6`       | IPv6 addresses in server records      |
| 1   | `0x0002` | `FEAT_QUERY`      | Search and pagination support         |
| 2   | `0x0004` | `FEAT_CLIENT_AUTH` | Tracker requires client authentication |
| 3   | `0x0008` | `FEAT_REG_ACK`    | Tracker supports registration acks    |
| 4   | `0x0010` | `FEAT_HMAC`       | Tracker supports HMAC-signed registration |

### Address Type Bytes (v3 Server Records)

| Value  | Name     | Address Size | Description         |
|--------|----------|-------------|---------------------|
| `0x04` | IPv4     | 4 bytes     | IPv4 address        |
| `0x06` | IPv6     | 16 bytes    | IPv6 address        |
| `0x48` | Hostname | 2+N bytes   | Length-prefixed hostname |

### Registration Ack Status Codes

| Code   | Name          | Description                                |
|--------|---------------|--------------------------------------------|
| `0x00` | `ACK_OK`      | Registration accepted                      |
| `0x01` | `ACK_DENIED`  | Password or authentication failed          |
| `0x02` | `ACK_BANNED`  | Source IP is banned                        |
| `0x03` | `ACK_QUOTA`   | Per-IP server quota exceeded               |
| `0x04` | `ACK_FULL`    | Tracker at maximum capacity                |
| `0x05` | `ACK_INVALID` | Malformed datagram                         |
| `0xFF` | `ACK_ERROR`   | Generic error                              |

### Complete TLV Field ID Table

| ID       | Name               | Type   | Category         | Sent by        |
|----------|--------------------|--------|------------------|----------------|
| `0x0010` | `DEREGISTER`       | bool   | Core             | Server (reg)   |
| `0x0100` | `ADDRESS_IPV6`     | ipv6   | Address          | Server (reg)   |
| `0x0101` | `HOSTNAME`         | string | Address          | Server (reg)   |
| `0x0200` | `SERVER_SOFTWARE`  | string | Descriptive      | Server (reg)   |
| `0x0201` | `COUNTRY_CODE`     | string | Descriptive      | Server (reg)   |
| `0x0202` | `REGION`           | string | Descriptive      | Server (reg)   |
| `0x0203` | `LANGUAGE`         | string | Descriptive      | Server (reg)   |
| `0x0204` | `MAX_USERS`        | u16    | Descriptive      | Server (reg)   |
| `0x0205` | `MATURITY`         | u8     | Descriptive      | Server (reg)   |
| `0x0206` | `UPTIME`           | u32    | Descriptive      | Server (reg)   |
| `0x0207` | `RULES_URL`        | string | Descriptive      | Server (reg)   |
| `0x0208` | `BANNER_URL`       | string | Descriptive      | Server (reg)   |
| `0x0209` | `ICON_URL`         | string | Descriptive      | Server (reg)   |
| `0x0300` | `PROTOCOL_VERSION` | u16    | Capability       | Server (reg)   |
| `0x0301` | `SUPPORTS_HOPE`    | bool   | Capability       | Server (reg)   |
| `0x0302` | `SUPPORTS_TLS`     | bool   | Capability       | Server (reg)   |
| `0x0303` | `TLS_PORT`         | u16    | Capability       | Server (reg)   |
| `0x0310` | `TAGS`             | string | Capability       | Server (reg)   |
| `0x0450` | `NEWS_COUNT`       | u32    | Content Index    | Tracker/Server |
| `0x0451` | `MSGBOARD_COUNT`   | u32    | Content Index    | Tracker/Server |
| `0x0452` | `FILES_COUNT`      | u32    | Content Index    | Tracker/Server |
| `0x0453` | `TOTAL_FILE_SIZE`  | u32    | Content Index    | Tracker/Server |
| `0x0500` | `PRIVATE_LISTING`  | bool   | Privacy          | Server (reg)   |
| `0x0600` | `IS_PROMOTED`      | bool   | Tracker-injected | Tracker only   |
| `0x0601` | `FIRST_SEEN`       | u32    | Tracker-injected | Tracker only   |
| `0x0602` | `LAST_HEARTBEAT`   | u32    | Tracker-injected | Tracker only   |
| `0x0700` | `FED_TRACKER_ID`   | bytes  | Federation       | Tracker (fed)  |
| `0x0701` | `FED_TRACKER_NAME` | string | Federation       | Tracker (fed)  |
| `0x0702` | `FED_TRACKER_DESC` | string | Federation       | Tracker (fed)  |
| `0x0703` | `FED_TRACKER_CONTACT` | string | Federation    | Tracker (fed)  |
| `0x0704` | `FED_TRACKER_POLICY` | string | Federation     | Tracker (fed)  |
| `0x0710` | `FED_SIGNATURE`    | bytes  | Federation       | Tracker (fed)  |
| `0x0800` | `REG_TOKEN`        | bytes  | Security         | Tracker (ack)  |
| `0x0801` | `HMAC_SHA256`      | bytes  | Security         | Server (reg)   |
| `0x0802` | `NONCE`            | bytes  | Security         | Server (reg)   |
| `0x0810` | `ERROR_MSG`        | string | Security         | Tracker (ack)  |
| `0x0811` | `TRACKER_NAME`     | string | Security         | Tracker (ack)  |
| `0x0820` | `AUTH_LOGIN`       | string | Auth             | Client (list)  |
| `0x0821` | `AUTH_PASS`        | string | Auth             | Client (list)  |
| `0x1001` | `SEARCH_TEXT`      | string | Query            | Client (list)  |
| `0x1010` | `PAGE_OFFSET`      | u16    | Query            | Client (list)  |
| `0x1011` | `PAGE_LIMIT`       | u16    | Query            | Client (list)  |

---

## Companion Specifications

The following companion specifications extend v3 with optional capabilities. Each is documented separately and can be implemented independently. They share the v3 TLV format and field ID namespace.

| Specification | Document | Description | TLV Range |
|---------------|----------|-------------|-----------|
| Content Index | — | Content statistics (news, files, message boards) injected by a content indexing service | 0x0400–0x04FF |
| Federation | [tracker-federation.md](tracker-federation.md) | HTFD protocol for tracker-to-tracker server list exchange with trust and identity controls | 0x0700–0x07FF |
