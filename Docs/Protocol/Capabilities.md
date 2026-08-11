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
| 3 | `0x0008` | `CAPABILITY_INLINE_MEDIA` | Inline image attachments via server-validated upload/download transactions (handle-based) | See [Inline Media Extension](capabilities-inline-media.md) |
| 4 | `0x0010` | `CAPABILITY_CHAT_HISTORY` | Server-side chat history retrieval | See [Chat History Extension](capabilities-chat-history.md) |
| 5 | `0x0020` | `CAPABILITY_EXTENDED_PRIV` | Extended 128-bit `FieldUserAccess` (110) bitmap (provisional) | See [Extended Privilege Bitmap Extension](capabilities-extended-priv.md) |
| 6 | `0x0040` | `CAPABILITY_MESSAGING` | Instant messaging layer: roster, presence, IM, offline delivery, discovery, file transfer, call signaling (provisional) | See [Instant Messaging Extension](capabilities-messaging.md) |
| 7 | `0x0080` | `CAPABILITY_DIRECT_TRANSFER` | Optional peer-to-peer (hole-punched) file transfer for messaging; absence uses the server relay (provisional) | See [Instant Messaging Extension](capabilities-messaging.md) |
| 8 | `0x0100` | `CAPABILITY_MESSENGER_SESSION` | Declares a pure instant-messenger session (provisional). Only meaningful alongside bit 6; a server MUST NOT confirm bit 8 unless it also confirms bit 6. When confirmed, the session is hidden from the classic Hotline world: excluded from Get User Name List (300) replies, and its join/leave/away announcements (Notify Change/Delete User 301/302) are suppressed. Rationale: a pure messenger never participates in public chat or classic PM, so listing it only confuses regular users — the session appears to join but cannot be interacted with. It remains fully visible through the messaging layer (roster, presence, discovery). | See [Instant Messaging Extension](capabilities-messaging.md) |
| 9 | `0x0200` | `CAPABILITY_MODERN_DATES` | Client parses the modern date encoding (actual year + seconds since Jan 1 of that year). Absent, the server MUST serve the legacy Mac 1904 epoch encoding (year field `1904`, seconds = total elapsed seconds since Jan 1 1904 UTC). A server MUST NOT infer this bit from any other capability: a classic Mac client reads the 4-byte seconds field as total-seconds-since-1904 and ignores the year field, so a modern date renders every timestamp in 1904. Ungated — a server confirms the bit whenever the client advertises it. | See [Date Format Selection](#date-format-selection) below |
| 10–63 | — | *Reserved* | Available for future extensions | — |

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
| Inline media | `DATA_CAPABILITIES` bit 3 + server config (`AccessSendMedia` permission; optional `MediaGateway` for public-chat legacy fallback only) |
| Chat history | `DATA_CAPABILITIES` bit 4 + server config (`Enabled` for history persistence) |
| Extended privilege bitmap | `DATA_CAPABILITIES` bit 5 + server config (account store widened to 128 bits) |
| Instant messaging | `DATA_CAPABILITIES` bit 6 (+ optional bit 7 for direct transfer) + server config (`Messaging.Enabled`) + `AccessMessaging` (bit 58) permission |
| HOPE secure login | Dedicated HOPE field IDs (0x0E01–0x0E04, 0x0EC1–0x0ECA) |
| Colored nicknames | Implicit opt-in (client sends `DATA_COLOR` in Set Client User Info) |
| GIF icons | No negotiation — feature is always available if server supports it |

`DATA_CAPABILITIES` is designed for features that require **explicit mutual agreement** before altered wire behaviour. Features that are purely additive (extra fields that can be safely ignored) may use simpler mechanisms.

---

## Implementation Notes

- **Server-side overrides:** The server may grant capabilities beyond what the client advertises. For example, a server may enable large-file mode for known clients (via a whitelist) even if they did not set `CAPABILITY_LARGE_FILES`. Similarly, a global toggle can enable a capability for all clients. This does **not** extend to `CAPABILITY_MODERN_DATES`, which selects a wire format the client must already be able to parse; see [Date Format Selection](#date-format-selection).
- **Bit width:** While currently bits 0–9 are defined, implementations should use a width that accommodates future growth. An 8-byte (64-bit) field provides 64 capability slots.
- **Unknown bits:** Both client and server should ignore bits they do not recognise. Do not reject a connection because of unknown capability bits.
- **Absence handling:** If `DATA_CAPABILITIES` is not present in the login request, the server should treat the client as having zero capabilities. If not present in the reply, the client should treat the session as standard mode.

---

## Date Format Selection

The Hotline protocol's 8-byte date structure (`year:2 | msecs:2 | secs:4`) has two competing wire encodings:

| Format | Year field | Seconds field | Used by |
|---|---|---|---|
| **Mac 1904 epoch** | `1904` | Total seconds since Jan 1, 1904 UTC | Vintage servers, vintage Mac clients |
| **Modern** | Actual year (e.g. `2026`) | Seconds since Jan 1 of that year, local time | Mobius, Hotline Navigator, most modern clients |

The two cannot be told apart by inspection: a legacy value is simply one whose year field happens to read `1904`. The encoding therefore has to be declared, not detected.

Many clients handle both formats transparently (e.g. hotline-1.0beta28's `Date(year:seconds:)` initializer works with either by accident of date arithmetic). However, some clients interpret the fields literally, and a mismatch in either direction produces a wrong date:

- **Legacy client, modern encoding.** The client reads the seconds field as total seconds since 1904 and ignores the year field. A seconds-since-Jan-1 offset (0 … ~31.5M) lands a few months into 1904, so dates render with roughly the right month and day and the year `1904` — one day out for dates after February in a non-leap year, because 1904 was a leap year.
- **Modern client, legacy encoding.** The client reads the year field as the calendar year and gets `1904`, then applies a seconds count far larger than a year. Depending on the width of its arithmetic, the result is a nonsensical date or an overflow.

### Server Behaviour

A server selects the encoding **per client**, from `CAPABILITY_MODERN_DATES` (bit 9) alone:

| Client advertises bit 9? | Server sends dates in… |
|---|---|
| **Yes** | Modern format (actual year + secs from Jan 1) |
| **No** (bit clear, or `DATA_CAPABILITIES` absent entirely) | Mac 1904 epoch format (year=1904, total secs since 1904) |

A server **MUST NOT** infer the modern encoding from any other capability, from the mere presence of `DATA_CAPABILITIES`, or from the client's version number. A client can want 64-bit file sizes or UTF-8 text while still reading dates the way an old Mac would, and a client under active development can change what it advertises between builds.

Because the choice is a wire format rather than a feature, negotiation is **ungated**: a server confirms bit 9 whenever the client advertises it, and has no reason to withhold it. This is a deliberate exception to the server-side overrides described under [Implementation Notes](#implementation-notes) — a server may grant *other* capabilities the client did not ask for, but must never grant this one.

**Range limit.** The legacy encoding carries an unsigned 32-bit second count from 1904, so it cannot express an instant after **2040-02-06 06:28:15 UTC**. A server encoding a later date MUST clamp to that ceiling rather than allow the conversion to wrap; a wrapped value re-enters 1904 and reproduces the symptom above. Clients that never advertise bit 9 will therefore see dates pinned at the ceiling after that point — visibly wrong, but never wrong in a way that looks plausible.

This applies to **all** date fields sent to the client:

- `FieldFileCreateDate` / `FieldFileModifyDate` in Get File Info (200) replies
- `CreateDate` / `ModifyDate` in `FlatFileInformationFork` during file and folder downloads
- `Date` in threaded news article listings and article data

The selection applies uniformly. Bit 9 is a statement about the client's decoder, not about one screen, so a server that honoured it for news but not for file dates would be sending one session two encodings — and the client has no way to tell which field carries which. Threaded news is merely where a wrong year is most obvious at a glance, which is why it is usually the first place the mismatch is reported.

### Client Behaviour

Clients should be prepared to handle both date formats. A robust decoder can distinguish the two by inspecting the year field:

- If `year == 1904`, seconds are total seconds since Jan 1, 1904 UTC.
- Otherwise, seconds are relative to Jan 1 of the given year.

A client that decodes both formats this way should still advertise bit 9 when it can: the legacy encoding it would otherwise receive cannot represent a date after February 2040.

A client that reads **only** the legacy encoding — which is the natural implementation on a classic Mac, and correct for one — must simply leave bit 9 clear. It then keeps receiving the encoding it understands no matter which other capabilities it negotiates, and needs no further changes. Bit 9 should be set in the same change that adds a modern-capable decoder, never ahead of it.

Clients that advertise bit 9 and see it confirmed in the login reply can expect dates in the modern format. A client that advertises it and does **not** see it echoed back is talking to a server that predates this bit; it will receive legacy dates and should decode accordingly.

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
