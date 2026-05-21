# Hotline Info Port — Capability Discovery Protocol

The **Hotline Info Port** is an optional, out-of-band TCP listener that lets a client discover what a Hotline server supports *before* opening a real session on the data port. It is purely advisory and has no effect on the legacy Hotline wire protocol — clients that don't know about the info port connect to the data port as they always have.

This document specifies the wire protocol so that any client or server can interoperate. It is intentionally minimal and self-describing: a fixed 8-byte header followed by a JSON body.

## Table of Contents

- [Goals and Non-Goals](#goals-and-non-goals)
- [Port Convention](#port-convention)
- [Wire Protocol](#wire-protocol)
  - [Request](#request-client--server)
  - [Response](#response-server--client)
  - [Status Codes](#status-codes)
- [Descriptor Schema (v1)](#descriptor-schema-v1)
- [Client Behavior](#client-behavior)
- [Server Behavior](#server-behavior)
- [Security and Trust Model](#security-and-trust-model)
- [Versioning](#versioning)
- [Worked Examples](#worked-examples)

---

## Goals and Non-Goals

**Goals**

- Let a supported client learn a server's transport (HOPE / TLS / plaintext), extensions (large files, voice, inline media, etc.), and policy hints (max users, banner mode, agreement size) without opening a Hotline session.
- Be trivially implementable in any language — no Hotline-specific framing beyond a fixed magic header.
- Be backwards-compatible: clients that don't speak it just connect to the data port; servers that don't speak it simply don't have the listener and the probe times out.
- Be operator-configurable: enable/disable, redact fields, change the bind port.

**Non-Goals**

- The info port does **not** authenticate, authorize, or transfer files.
- It is **not** a substitute for a tracker. A tracker advertises *which* servers exist; the info port describes *what one specific server offers*.
- It is **not** a replacement for the Hotline login handshake. Real session negotiation (HOPE algorithm choice, capability bits, etc.) still happens on the data port.

---

## Port Convention

The default info port is **one below the data port**:

| Data port (`Port`) | Default info port |
|---|---|
| 5500 (standard) | **5499** |
| 6000 | 5999 |

This is a convention, not a requirement — the server may bind any port the operator chooses. A client that doesn't know the configured port should default to `Port - 1`.

The data port itself (`Port`) and the file-transfer port (`Port + 1`) are unchanged. A typical Janus-style server therefore occupies a contiguous trio:

```
Port - 1   info (this protocol)
Port       Hotline data / transactions
Port + 1   Hotline file transfers
```

---

## Wire Protocol

All multi-byte integers are **big-endian**. All offsets are in bytes from the start of the frame.

### Request (client → server)

The client opens a TCP connection and writes exactly **8 bytes**:

| Offset | Size | Field          | Value                                              |
|-------:|-----:|----------------|----------------------------------------------------|
|      0 |    4 | `magic`        | ASCII `"HLIP"` (`0x48 0x4C 0x49 0x50`)             |
|      4 |    2 | `version`      | Info-protocol version requested (current: `1`)     |
|      6 |    2 | `flags`        | Reserved. MUST be `0` in v1.                       |

The server SHOULD apply a read deadline (5 seconds is plenty) and SHOULD silently close the connection if the magic does not match.

### Response (server → client)

The server writes a **12-byte header** followed by an optional **JSON payload**:

| Offset | Size | Field           | Value                                                  |
|-------:|-----:|-----------------|--------------------------------------------------------|
|      0 |    4 | `magic`         | ASCII `"HLIP"`                                         |
|      4 |    2 | `version`       | Info-protocol version of this response                 |
|      6 |    2 | `status`        | Status code (see below)                                |
|      8 |    4 | `payloadLength` | Length in bytes of the JSON payload that follows       |
|     12 |    N | `payload`       | UTF-8 JSON object. Length = `payloadLength`            |

- `payloadLength` MAY be `0` (e.g. for error statuses), in which case no body follows.
- The server SHOULD close the connection immediately after writing the response. The info exchange is one-shot — clients re-probing should open a new connection.
- A `payloadLength` greater than a sane cap (suggested: 64 KiB) MUST be treated as an error by the client.

### Status Codes

| Code | Name                    | Meaning |
|-----:|-------------------------|---------|
|    0 | `OK`                    | Descriptor follows in the payload.                                    |
|    1 | `UNSUPPORTED_VERSION`   | The client requested a version the server does not implement. The `version` field in the response is the highest version the server supports — the client MAY retry. |
|    2 | `DISABLED`              | The info port is administratively disabled. (Reserved — typically the server simply does not listen at all.) |
|    3 | `INTERNAL_ERROR`        | Server-side failure building the descriptor. The client SHOULD treat this the same as no response.            |

Implementations MAY define additional codes ≥ 256; codes below 256 are reserved for this specification.

---

## Descriptor Schema (v1)

The payload is a single JSON object. All fields are OPTIONAL except `infoVersion`, `dataPort`, and `transport`. Unknown fields MUST be ignored by clients (forward compatibility).

```jsonc
{
  // REQUIRED. Always matches the version in the response header.
  "infoVersion": 1,

  // OPTIONAL. Short opaque hash of the descriptor's material fields.
  // Clients SHOULD cache the full descriptor keyed by (host, port) and
  // skip re-fetching while this value is unchanged.
  "etag": "a1b2c3d4e5f60718",

  // OPTIONAL. Server software fingerprint. Operators MAY redact this.
  "software": {
    "name": "janus",
    "version": "2.6.0"
  },

  // OPTIONAL. Human-readable identity. Fields mirror the tracker
  // registration data but are not authoritative for tracker listings.
  "server": {
    "name": "Example Hotline",
    "description": "A friendly community server",
    "hostname": "hotline.example.com"
  },

  // REQUIRED. The data port to connect to for the real Hotline session.
  // Almost always the same TCP port the client just probed + 1, but the
  // server is authoritative — clients MUST use this value, not assume.
  "dataPort": 5500,

  // OPTIONAL. Present when the server has a TLS listener.
  "tlsPort": 5600,

  // REQUIRED. Per-transport capability matrix.
  "transport": {
    "hope": {
      "supported": true,   // server can perform HOPE secure login
      "required":  false   // server REJECTS plaintext / non-HOPE sessions
    },
    "tls": {
      "supported": true,   // server has a TLS listener
      "required":  false   // server rejects non-TLS sessions
    },
    "plaintext": {
      "accepted": true     // server accepts unencrypted Hotline sessions
    },
    "compression": {
      "supported": true    // HOPE transport compression available
    }
  },

  // OPTIONAL. Protocol-extension flags. Each key is a boolean; absent
  // keys mean "not supported". Clients MUST NOT infer support for an
  // extension from any other field.
  "capabilities": {
    "largeFiles":   true,   // 64-bit file sizes
    "inlineMedia":  true,   // inline-image / media extension
    "voice":        false,  // WebRTC voice chat
    "chatHistory":  true,   // server-side chat history / replay
    "coloredNicks": true    // colored-nickname extension
  },

  // OPTIONAL. Banner mode. "mode" is one of "local", "remote", "none".
  "banner": {
    "mode": "local",
    "clickURL": "https://example.com/"
  },

  // OPTIONAL. Server agreement metadata. Present only when an agreement
  // is configured. The agreement text itself is NOT included — clients
  // fetch it during the login handshake as normal.
  "agreement": {
    "sizeBytes": 1024
  },

  // OPTIONAL. Server policy limits relevant to connect planning.
  "limits": {
    "maxClients":          100,
    "maxConnectionsPerIP": 5
  },

  // OPTIONAL. Live user count. Operators MAY suppress this.
  "users": {
    "connected": 42
  }
}
```

### Field Semantics — Notes

- **`transport.hope.required`** + **`transport.tls.required`** are *server-side enforcement* signals. If `hope.required` is `true`, the server will reject a plaintext login on the data port. Clients SHOULD honor this in their connect plan rather than discover it the hard way.
- **`transport.plaintext.accepted`** is the inverse of "encryption is mandatory."
- **`capabilities.*`** flags describe what the *server* can do. The client still negotiates the actual session capabilities during the Hotline login handshake; the info port is a hint, not a contract.
- **`etag`** is opaque. Clients SHOULD NOT parse or interpret it; only compare for equality.

---

## Client Behavior

The recommended flow for a "capability-aware" client connecting to a server with a saved bookmark or freshly-entered host:

1. **Check cache.** If a descriptor for `(host, infoPort)` is cached with an unexpired TTL, use it. Skip to step 5.
2. **Probe the info port.** Open TCP to `host : infoPort` (default `dataPort - 1`). Send the 8-byte request frame. Apply a short read deadline (300–800 ms is typical).
3. **Handle the response:**
   - `status = OK` → parse the JSON descriptor; update cache keyed by `(host, infoPort)` with the `etag`.
   - `status = UNSUPPORTED_VERSION` → optionally retry at the version reported in the response header.
   - **Timeout / connection refused / EOF** → assume the server does not speak the info protocol. Add `(host, infoPort)` to a **negative cache** for a tunable duration (suggested: 1 hour to 1 day) so the client doesn't probe on every connect.
   - Malformed response → treat as no response.
4. **Adjust the connect plan.** Apply the discovered transport settings: enable HOPE if available, prefer TLS if required, etc. **See Security and Trust Model below — a descriptor MUST NOT cause the client to silently weaken a saved security setting.**
5. **Open the real session** on the data port using the adjusted plan.

A client MAY skip the probe entirely for bookmarks marked "info disabled" by the user.

The probe SHOULD NOT block UI for long. Run it concurrently with any UI feedback and surface a clear error if the data port then refuses the chosen transport.

---

## Server Behavior

A conforming server:

1. **Listens** on the configured info port (default `dataPort - 1`) when the feature is enabled.
2. **Accepts** TCP connections, applies a read deadline, and reads exactly 8 bytes.
3. **Rejects unknown magic** by closing the connection silently. This is normal traffic from port scanners; do not log loudly.
4. **Replies** with the 12-byte header and (on `OK`) a JSON descriptor, then closes the connection.
5. **Caps payload size** (recommended: 64 KiB) to bound response cost.
6. **Rate-limits / firewalls** the info port independently of the data port. The info port is cheaper than a full login, so its rate limit can be looser, but it should still be defended against probe floods.
7. **Handles version skew**: if the client requests `version > server-max`, reply with `UNSUPPORTED_VERSION` and put the server's max version in the response header.

The info port MUST NOT consume a session slot, MUST NOT count against the data port's per-IP connection limit, and MUST NOT trigger an auto-ban from the data port's connect-rate limiter. (If you want to defend the info port separately, use a dedicated info-port rate limiter.)

The info port MAY be served over plaintext TCP even when the data port requires HOPE/TLS — capability advertisement is by nature public, and the descriptor itself signals to the client that the real session needs encryption.

---

## Security and Trust Model

The info port is **plaintext by design**. An attacker positioned to MITM the info port can lie about server capabilities — most dangerously, by claiming "no HOPE, no TLS, plaintext OK" and harvesting credentials.

Clients MUST follow this rule:

> **A descriptor may strengthen security settings; it must not silently weaken them.**

Concretely:

- If a saved bookmark says "use HOPE" and the descriptor says HOPE is unavailable, the client MUST refuse the connection with a visible error, NOT silently downgrade to plaintext.
- If the descriptor says HOPE is required and the user has not configured HOPE credentials, the client MUST surface this to the user before attempting the data port.
- For first contact (no saved settings), the client SHOULD prefer the strictest transport the descriptor advertises as supported.

Operators who treat capability advertisement as sensitive MAY:

- Disable the info port entirely (`Enabled: false` / equivalent).
- Redact the software fingerprint (`IncludeSoftware: false`).
- Suppress the live user count (`IncludeUserCount: false`).
- Place the info port behind a firewall / VPN.

Future versions of this protocol MAY introduce signed descriptors. Implementations SHOULD treat the wire format as evolvable through the `infoVersion` field.

---

## Versioning

- The `version` field in the request and response headers is the **wire protocol** version, currently `1`.
- The `infoVersion` field in the JSON body equals the response header's `version` — it is duplicated for the benefit of tools that consume the payload without seeing the header.
- The JSON body itself is forward-extensible: clients MUST ignore unknown fields.
- Breaking changes to the JSON schema (e.g. renaming a field) MUST bump the wire version.

---

## Worked Examples

### Example 1 — Minimum-information server

A small private server that has disabled the software fingerprint and user count.

**Request** (8 bytes, hex):

```
48 4C 49 50  00 01  00 00
H  L  I  P   v=1    flags
```

**Response header** (12 bytes, hex):

```
48 4C 49 50  00 01  00 00  00 00 01 0F
H  L  I  P   v=1    OK     payloadLength=271
```

**Payload** (JSON, 271 bytes):

```json
{
  "infoVersion": 1,
  "etag": "82c1d5f4a8b03326",
  "dataPort": 5500,
  "transport": {
    "hope":        {"supported": true,  "required": true},
    "tls":         {"supported": false},
    "plaintext":   {"accepted": false},
    "compression": {"supported": true}
  },
  "capabilities": {"largeFiles": true}
}
```

A client seeing this would: enable HOPE, refuse to attempt plaintext, and connect to `host:5500`.

### Example 2 — Fully populated descriptor

```json
{
  "infoVersion": 1,
  "etag": "f7301a9b2c4d5e60",
  "software": {"name": "janus", "version": "2.6.0"},
  "server": {
    "name": "Big Red H",
    "description": "Community Hotline server, est. 2021",
    "hostname": "bigredh.com"
  },
  "dataPort": 5500,
  "tlsPort": 5600,
  "transport": {
    "hope":        {"supported": true,  "required": false},
    "tls":         {"supported": true,  "required": false},
    "plaintext":   {"accepted": true},
    "compression": {"supported": true}
  },
  "capabilities": {
    "largeFiles":   true,
    "inlineMedia":  true,
    "voice":        false,
    "chatHistory":  true,
    "coloredNicks": true
  },
  "banner": {"mode": "local", "clickURL": "https://bigredh.com/"},
  "agreement": {"sizeBytes": 1024},
  "limits": {"maxClients": 100, "maxConnectionsPerIP": 5},
  "users": {"connected": 12}
}
```

### Example 3 — Server doesn't speak the info protocol

Client opens TCP to `host:5499`, sends `HLIP 00 01 00 00`. Server either:

- Refuses the connection (port not listening).
- Closes the connection without reading.
- Times out.

Client treats any of these as "no info protocol available," adds the host/port to its negative cache, and falls back to its saved transport settings for the data port connection.

### Example 4 — Version skew

Future client requests `version = 2`. Server only implements `version = 1`.

**Response header**:

```
48 4C 49 50  00 01  00 01  00 00 00 00
H  L  I  P   v=1    UNSUP  payloadLength=0
```

The client sees status `1` (UNSUPPORTED_VERSION) and the response `version = 1`. It MAY retry the request with `version = 1`.

---

## Reference Implementation

A new client implementation should be testable against any conforming server using nothing more than a TCP socket and a JSON parser — no Hotline-specific code is required to consume the descriptor.
