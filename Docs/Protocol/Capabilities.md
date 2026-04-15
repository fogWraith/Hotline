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
- [Date Format Selection](#date-format-selection)
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
| 1 | `0x0002` | `CAPABILITY_TEXT_ENCODING` | UTF-8 text encoding for all string data | See [Text Encoding Extension](capabilities-text-encoding.md) |
| 2 | `0x0004` | `CAPABILITY_VOICE` | Voice chat via WebRTC SFU | See [Voice Chat Extension](capabilities-voice.md) |
| 3 | `0x0008` | `CAPABILITY_INLINE_MEDIA` | Inline image attachments in chat | See [Inline Media Extension](capabilities-inline-media.md) |
| 4 | `0x0010` | `CAPABILITY_CHAT_HISTORY` | Server-side chat history retrieval | See [Chat History Extension](https://github.com/fuzzywalrus/lemoniscate/blob/main/docs/Capabilities-Chat-History.md) |
| 5–63 | — | *Reserved* | Available for future extensions | — |

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
| Text encoding | `DATA_CAPABILITIES` bit 1 + server config fallback |
| Voice chat | `DATA_CAPABILITIES` bit 2 + server config (`EnableVoice`) |
| Inline media | `DATA_CAPABILITIES` bit 3 + server config (`MediaGateway` for legacy fallback) |
| Chat history | `DATA_CAPABILITIES` bit 4 + server config (`Enabled` for history persistence) |
| HOPE secure login | Dedicated HOPE field IDs (0x0E01–0x0E04, 0x0EC1–0x0ECA) |
| Colored nicknames | Implicit opt-in (client sends `DATA_COLOR` in Set Client User Info) |
| GIF icons | No negotiation — feature is always available if server supports it |

`DATA_CAPABILITIES` is designed for features that require **explicit mutual agreement** before altered wire behaviour. Features that are purely additive (extra fields that can be safely ignored) may use simpler mechanisms.

---

## Implementation Notes

- **Server-side overrides:** The server may grant capabilities beyond what the client advertises. For example, a server may enable large-file mode for known clients (via a whitelist) even if they did not set `CAPABILITY_LARGE_FILES`. Similarly, a global toggle can enable a capability for all clients.
- **Bit width:** While currently bits 0–1 are defined, implementations should use a width that accommodates future growth. An 8-byte (64-bit) field provides 64 capability slots.
- **Unknown bits:** Both client and server should ignore bits they do not recognise. Do not reject a connection because of unknown capability bits.
- **Absence handling:** If `DATA_CAPABILITIES` is not present in the login request, the server should treat the client as having zero capabilities. If not present in the reply, the client should treat the session as standard mode.

---

## Date Format Selection

The Hotline protocol's 8-byte date structure (`year:2 | msecs:2 | secs:4`) has two competing wire encodings:

| Format | Year field | Seconds field | Used by |
|---|---|---|---|
| **Mac 1904 epoch** | `1904` | Total seconds since Jan 1, 1904 UTC | Vintage servers, vintage Mac clients |
| **Modern** | Actual year (e.g. `2026`) | Seconds since Jan 1 of that year, local time | Mobius, Hotline Navigator, most modern clients |

Many clients handle both formats transparently (e.g. hotline-1.0beta28's `Date(year:seconds:)` initializer works with either by accident of date arithmetic). However, some clients interpret the fields literally — if they receive year=1904 but treat the seconds field as relative to that year, the decoded date overflows into nonsensical values.

### Server Behaviour

Servers that support `DATA_CAPABILITIES` use the field's presence as a signal to select the appropriate date encoding **per client**:

| Client sends `DATA_CAPABILITIES`? | Server sends dates in… |
|---|---|
| **Yes** (any non-zero value) | Modern format (actual year + secs from Jan 1) |
| **No** (field absent) | Mac 1904 epoch format (year=1904, total secs since 1904) |

This applies to **all** date fields sent to the client:

- `FieldFileCreateDate` / `FieldFileModifyDate` in Get File Info (200) replies
- `CreateDate` / `ModifyDate` in `FlatFileInformationFork` during file and folder downloads
- `Date` in threaded news article listings and article data

The issue has been most prominent in threaded news articles, date fields in Get File Info (200) and file/folder downloads may or may not be required, but better safe than sorry. One could probably get away with only addressing Date in threaded news.

### Client Behaviour

Clients should be prepared to handle both date formats. A robust decoder can distinguish the two by inspecting the year field:

- If `year == 1904`, seconds are total seconds since Jan 1, 1904 UTC.
- Otherwise, seconds are relative to Jan 1 of the given year.

Clients that send `DATA_CAPABILITIES` during login can expect dates in the modern format.

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
