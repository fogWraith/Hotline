# DATA_CAPABILITIES – Session Capability Negotiation

`DATA_CAPABILITIES` is a field used during the Hotline login sequence to negotiate protocol extensions between client and server. Each extension is represented by a single bit in a bitmask, allowing both sides to advertise and confirm support for specific features.

## Table of Contents

- [Overview](#overview)
- [Field Definition](#field-definition)
- [Negotiation Flow](#negotiation-flow)
- [Defined Capability Bits](#defined-capability-bits)
- [Usage in Login (107)](#usage-in-login-107)
- [Interaction with Other Negotiation Mechanisms](#interaction-with-other-negotiation-mechanisms)
- [Implementation Notes](#implementation-notes)
- [Example](#example)

---

## Overview

The standard Hotline protocol provides no general mechanism for clients and servers to negotiate feature support beyond the 16-bit version number. Extensions like HOPE use their own field IDs for negotiation, and others like colored nicknames rely on implicit detection.

`DATA_CAPABILITIES` provides an explicit, bitmask-based negotiation channel. Clients advertise which extensions they support, and the server confirms which of those are enabled for the session by echoing the relevant bits back.

---

## Field Definition

| Property | Value |
|---|---|
| Field ID | `0x01F0` (496 decimal) |
| Name | `DATA_CAPABILITIES` |
| Type | Unsigned integer (big-endian) |
| Width | Variable; typically 2 bytes, expandable to 8 bytes (64-bit) |

The field is a bitmask. Each bit represents a specific protocol extension. Unrecognised bits should be ignored by both sides.

---

## Negotiation Flow

1. **Client → Server (Login request):** The client includes `DATA_CAPABILITIES` in its Login (107) packet with bits set for each extension it supports.

2. **Server processing:** The server examines the advertised capabilities alongside its own configuration. For each capability, the server decides whether to enable it for this session based on the client's advertisement and server-side policy (e.g. the server may grant large-file support via a whitelist even if the client did not advertise it).

3. **Server → Client (Login reply):** The server includes `DATA_CAPABILITIES` in the login reply with bits set for each extension that is **active** for this session.

4. **Session behaviour:** Both sides use the negotiated capabilities to determine which protocol features are available for the duration of the connection.

If `DATA_CAPABILITIES` is absent from either the request or reply, no extension features are active — the session operates in standard Hotline mode.

---

## Defined Capability Bits

| Bit | Mask | Name | Description | Specification |
|---|---|---|---|---|
| 0 | `0x0001` | `CAPABILITY_LARGE_FILES` | 64-bit file sizes and transfer lengths | See [Large File Extension](capabilities-large-file.md) |
| 1–63 | — | *Reserved* | Available for future extensions | — |

---

## Usage in Login (107)

### Client Request

The client includes `DATA_CAPABILITIES` alongside the standard login fields:

| ID | Field | Notes |
|---|---|---|
| 105 | User login | |
| 106 | User password | |
| 160 | Version | Client protocol version |
| 102 | User name | |
| 104 | User icon ID | |
| **0x01F0** | **Capabilities** | **Bitmask of supported extensions** |

### Server Reply

The server includes `DATA_CAPABILITIES` in the login reply to confirm which extensions are active:

| ID | Field | Notes |
|---|---|---|
| 0x00A2 | Server name | |
| 160 | Version | Server protocol version |
| 103 | User ID | Assigned UID |
| **0x01F0** | **Capabilities** | **Bitmask of confirmed extensions** |

If the server does not support any of the client's advertised capabilities, `DATA_CAPABILITIES` may be omitted from the reply entirely — the session falls back to standard mode. Clients should treat an absent `DATA_CAPABILITIES` field in the reply as a zero bitmask.

---

## Interaction with Other Negotiation Mechanisms

`DATA_CAPABILITIES` complements other extension negotiation approaches:

| Extension | Negotiation Mechanism |
|---|---|
| Large files | `DATA_CAPABILITIES` bit 0 + server config overrides |
| HOPE secure login | Dedicated HOPE field IDs (0x0E01–0x0E04, 0x0EC1–0x0ECA) |
| Colored nicknames | Implicit opt-in (client sends `DATA_COLOR` in Set Client User Info) |
| GIF icons | No negotiation — feature is always available if server supports it |

`DATA_CAPABILITIES` is designed for features that require **explicit mutual agreement** before altered wire behaviour. Features that are purely additive (extra fields that can be safely ignored) may use simpler mechanisms.

---

## Implementation Notes

- **Server-side overrides:** The server may grant capabilities beyond what the client advertises. For example, a server may enable large-file mode for known clients (via a whitelist) even if they did not set `CAPABILITY_LARGE_FILES`. Similarly, a global toggle can enable a capability for all clients.
- **Bit width:** While currently only bit 0 is defined, implementations should use a width that accommodates future growth. An 8-byte (64-bit) field provides 64 capability slots.
- **Unknown bits:** Both client and server should ignore bits they do not recognise. Do not reject a connection because of unknown capability bits.
- **Absence handling:** If `DATA_CAPABILITIES` is not present in the login request, the server should treat the client as having zero capabilities. If not present in the reply, the client should treat the session as standard mode.

---

## Example

A client advertises `CAPABILITY_LARGE_FILES` and the server confirms it:

**Client → Server (Login 107):**

```
Field: DATA_CAPABILITIES (0x01F0)
Length: 2
Value:  00 01    ← bit 0 set (CAPABILITY_LARGE_FILES)
```

**Server → Client (Login Reply):**

```
Field: DATA_CAPABILITIES (0x01F0)
Length: 2
Value:  00 01    ← bit 0 confirmed
```

Large-file mode is now active for the session. Subsequent file transactions will include 64-bit companion fields, and HTXF transfers will use the large-file flag.

If the server did not support or denied large files:

```
Field: DATA_CAPABILITIES — absent from reply (or value 00 00)
```

The session operates in standard 32-bit mode.
