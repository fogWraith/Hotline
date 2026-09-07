# Building a Hotline Instant Messenger Client

> Last updated: August 9, 2026 - tracks the spec as of the same date.
> Status: implementation guide (informative). The normative wire
> definitions live in
> [Instant Messaging Extension](../Protocol/Capabilities-Messaging.md)
> and the base [Hotline Protocol](../Protocol/Hotline.md); where this
> guide and a normative spec disagree, the spec wins.

This document is a language-agnostic, end-to-end guide for building a **pure
instant-messenger client** that speaks the Hotline [Instant Messaging
Extension](../Protocol/Capabilities-Messaging.md). It is written so
that a developer with no prior Hotline knowledge can implement a fully
interoperable client in any language with a TCP socket, a few hash primitives,
and (for the modern profile) a WebRTC stack.

It deliberately covers the whole stack a messenger needs - transport, framing,
login, capability negotiation, text encoding, and every messaging flow - rather
than assuming a pre-existing Hotline file-sharing client. A *pure messenger*
ignores files, news, and public chat; it negotiates only the capabilities it
needs and shows the user a buddy list, conversations, presence, file sends, and
(optionally) calls.

## How to read this guide

The guide defines **two client profiles** and flags every divergence between
them:

- **Legacy profile** - targets vintage operating systems (Windows 95,
  Macintosh System 7) and constrained runtimes. Plaintext or HOPE-stream
  transport, Mac Roman text, relay-only file transfer, **no** voice. The
  smallest thing that is a conforming messenger.
- **Modern profile** - targets contemporary systems (macOS 15+, current
  Windows/Linux). HOPE AEAD or TLS transport, UTF-8 text, info-port discovery,
  direct (hole-punched) file transfer, and voice/conference calls over WebRTC.

Sections marked **[Legacy]**, **[Modern]**, or **[Both]** tell you which profile
they apply to. If a section is unmarked, it applies to both.

---

## Table of Contents

1. [Conventions](#1-conventions)
2. [Architecture you must internalise](#2-architecture-you-must-internalise)
3. [The two client profiles at a glance](#3-the-two-client-profiles-at-a-glance)
4. [Transport layer](#4-transport-layer)
   - 4.1 [Port layout](#41-port-layout)
   - 4.2 [Info-port discovery (Modern)](#42-info-port-discovery-modern)
   - 4.3 [TCP handshake (TRTP)](#43-tcp-handshake-trtp)
   - 4.4 [TLS transport (Modern)](#44-tls-transport-modern)
   - 4.5 [Choosing a transport](#45-choosing-a-transport)
5. [Wire framing primitives](#5-wire-framing-primitives)
   - 5.1 [Transaction header](#51-transaction-header)
   - 5.2 [Field list and field encoding](#52-field-list-and-field-encoding)
   - 5.3 [Integer width rules](#53-integer-width-rules)
   - 5.4 [Task IDs, replies, and notifications](#54-task-ids-replies-and-notifications)
   - 5.5 [Repeated-entry (list) encoding](#55-repeated-entry-list-encoding)
6. [Text encoding](#6-text-encoding)
7. [Login and capability negotiation](#7-login-and-capability-negotiation)
   - 7.1 [Capability bitmask](#71-capability-bitmask)
   - 7.2 [Legacy plaintext login](#72-legacy-plaintext-login)
   - 7.3 [HOPE secure login](#73-hope-secure-login)
   - 7.4 [Reading the login reply](#74-reading-the-login-reply)
   - 7.5 [Post-login: the pure-messenger shortcut](#75-post-login-the-pure-messenger-shortcut)
   - 7.6 [ChaCha20-Poly1305 AEAD wire format (Modern)](#76-chacha20-poly1305-aead-wire-format-modern)
8. [Messaging object & transaction reference](#8-messaging-object--transaction-reference)
9. [Roster and friend management](#9-roster-and-friend-management)
   - 9.1 [Roster sync on login - Get Roster (800)](#91-roster-sync-on-login--get-roster-800)
   - 9.2 [Live deltas - Roster Entry (801)](#92-live-deltas--roster-entry-801)
   - 9.3 [Add Friend (802)](#93-add-friend-802)
   - 9.4 [Friend Request (804) and Friend Response (805)](#94-friend-request-804-and-friend-response-805)
   - 9.5 [Remove (803), Block (806), Unblock (807)](#95-remove-803-block-806-unblock-807)
   - 9.6 [Set Friend Nickname (824)](#96-set-friend-nickname-824)
10. [User discovery and profiles](#10-user-discovery-and-profiles)
    - 10.1 [Find User (822)](#101-find-user-822)
    - 10.2 [User Search (823)](#102-user-search-823)
    - 10.3 [User profiles (825 / 826)](#103-user-profiles-825--826)
11. [Presence](#11-presence)
    - 11.1 [Set Presence (808)](#111-set-presence-808)
    - 11.2 [Presence Changed (809)](#112-presence-changed-809)
12. [Instant messaging](#12-instant-messaging)
    - 12.1 [IM Send (810)](#121-im-send-810)
    - 12.2 [IM Deliver (811)](#122-im-deliver-811)
    - 12.3 [IM Acknowledge (812)](#123-im-acknowledge-812)
    - 12.4 [IM Typing (813)](#124-im-typing-813)
13. [Offline delivery](#13-offline-delivery)
14. [User-to-user file transfer](#14-user-to-user-file-transfer)
    - 14.1 [Signaling](#141-signaling)
    - 14.2 [Relay path](#142-relay-path-both--the-only-path-legacy-uses)
    - 14.3 [Direct path (Modern, opt-in)](#143-direct-path-modern-opt-in)
    - 14.4 [Transfer-port encryption](#144-transfer-port-encryption)
15. [Call signaling and voice (Modern)](#15-call-signaling-and-voice-modern)
    - 15.1 [Ring flow](#151-ring-flow)
    - 15.2 [Media](#152-media-summary--see-the-voice-spec-for-the-full-detail)
16. [Reason codes and error handling](#16-reason-codes-and-error-handling)
    - 16.1 [The shape of every reply](#161-the-shape-of-every-reply)
17. [Recommended client model](#17-recommended-client-model)
18. [Resilience: reconnect, idempotency, dedup](#18-resilience-reconnect-idempotency-dedup)
19. [Security considerations for client authors](#19-security-considerations-for-client-authors)
20. [Conformance checklist](#20-conformance-checklist)
- [Appendix A - Enumerations](#appendix-a--enumerations)
- [Appendix B - Field ID quick table](#appendix-b--field-id-quick-table)
- [Appendix C - Transaction ID quick table](#appendix-c--transaction-id-quick-table)
- [Appendix D - Worked byte-level example: IM Send](#appendix-d--worked-byte-level-example-im-send)

---

## 1. Conventions

- **Byte order is big-endian (network byte order) for every multi-byte integer**
  in every Hotline structure, on the data port, the transfer port, and the info
  port. The only JSON payloads (info-port descriptor, ICE candidates) are UTF-8
  text and follow JSON rules.
- Conformance keywords (MUST, SHOULD, MAY, …) are used as in
  [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).
- Notation: `u8`, `u16`, `u32`, `u64` are unsigned big-endian integers of that
  width. `bytes[n]` is a fixed-length opaque byte string. `str` is text in the
  session's negotiated encoding (see [§6](#6-text-encoding)).
- Field IDs are given in both decimal and hex. Transaction type IDs are decimal
  with hex in parentheses where useful. Hex literals are prefixed `0x`.
- "The spec" without qualification means the
  [Instant Messaging Extension](../Protocol/Capabilities-Messaging.md).

---

## 2. Architecture you must internalise

Three distinct identifiers exist. Confusing them is the single most common
implementation bug. (Source: spec, *Identity Model*.)

| Layer | Carried in | Stable across reconnect? | What it addresses |
|---|---|---|---|
| **Bare identity** - the Login | `DATA_FRIEND_LOGIN` (0x0600) | **Yes** | Roster membership, the friend graph, presence, message/request addressing. This is the public handle a user types to add a friend. |
| **Endpoint** - the 16-bit user ID | `FieldUserID` (103) | **No** - reassigned every login | Wire-level delivery target; the specific live session a call or file transfer connects to. |
| **Display name** - the nickname | `FieldUserName` (102) | No - mutable, non-unique | Cosmetic only. Never an identity. |

Consequences a client MUST respect:

1. **Address friends by Login, never by user ID.** You add `alice`, you message
   `alice`, you see presence for `alice`. The server resolves `alice` to her
   live session(s) at delivery time.
2. **A Login may have several live sessions at once** (e.g. the same person on a
   System 7 box and a phone). Text messages and presence fan out to *all* of a
   Login's sessions; calls/transfers ring all and the first to accept wins.
3. **The Login is a public handle**; security rests entirely on the password.
   Treat a friend's Login as you would an email address - knowable, shareable,
   not a secret.
4. **Presence is aggregated** server-side across a Login's sessions. You receive
   one presence value per friend, never one per session.

The server keeps an in-memory `Login → {sessions}` index and does all
friendship/block enforcement. **The client is never trusted** to enforce
authorization - but a good client mirrors the rules locally to keep the UI
honest (e.g. don't show a "call" button for a friend who isn't `Accepted`).

---

## 3. The two client profiles at a glance

| Concern | Legacy profile | Modern profile |
|---|---|---|
| Transport | Plaintext; HOPE authentication over a plaintext socket; or HOPE with an RC4/Blowfish stream cipher | HOPE with ChaCha20-Poly1305 AEAD (preferred) or TLS |
| Capability discovery | Skip; connect straight to the data port | [Info port](#42-info-port-discovery-modern) probe first |
| Text | Mac Roman (no `CAPABILITY_TEXT_ENCODING`) | UTF-8 (`CAPABILITY_TEXT_ENCODING`, bit 1) |
| Capability bits advertised | `MESSAGING` (6) + `MESSENGER_SESSION` (8) [+ `TEXT_ENCODING` if able] | `MESSAGING` (6) + `MESSENGER_SESSION` (8) + `TEXT_ENCODING` (1) + `VOICE` (2) + `DIRECT_TRANSFER` (7) [+ `LARGE_FILES` (0)] |
| Roster / presence / IM | Full | Full |
| File transfer | Relay path only | Relay + direct (hole-punched) with relay fallback |
| Voice / conference calls | Not supported | Supported (WebRTC SFU) |
| Compression | Optional (GZIP) | Optional (ZSTD/LZ4/GZIP) |

Both profiles share the **same messaging transactions and fields** - the
difference is transport, encoding, and which optional capabilities are turned
on. Build the legacy core first; the modern profile is additive.

---

## 4. Transport layer

### 4.1 Port layout

A Hotline server occupies a contiguous block based on a single **base port**
(default 5500):

| Port | Purpose |
|---|---|
| base − 1 | Info port (capability discovery), **Modern** |
| **base** | Data port - transactions (this is where messaging lives) |
| base + 1 | File-transfer port (HTXF), used by [§14](#14-user-to-user-file-transfer) |
| base + 4 | Voice media (UDP, WebRTC), used by [§15](#15-call-signaling-and-voice-modern) |

If the server has a TLS listener it is on a separate port advertised as
`tlsPort` (default 5600); its transfer port is `tlsPort + 1`.

### 4.2 Info-port discovery (Modern)

**[Modern]** Before opening a session, a modern client SHOULD probe the info
port to learn the server's transport requirements and extension support, so it
can pick the right transport without trial and error. Full spec:
[Hotline Info Port](../Protocol/Hotline-Info-Port.md).

Request - open TCP to `host:(base−1)` and write exactly 8 bytes:

```
"HLIP"  u16:version=1  u16:flags=0
48 4C 49 50  00 01  00 00
```

Response - a 12-byte header then a JSON body:

```
"HLIP"  u16:version  u16:status  u32:payloadLength   <payload: UTF-8 JSON>
```

`status == 0` (`OK`) means the JSON descriptor follows. The fields a messenger
cares about:

```jsonc
{
  "dataPort": 5500,
  "tlsPort": 5600,                      // present only if TLS available
  "transport": {
    "hope": { "supported": true, "required": false },
    "tls":  { "supported": true, "required": false },
    "plaintext": { "accepted": true },
    "compression": { "supported": true }
  },
  "capabilities": { "voice": true, "largeFiles": true }
}
```

Client rules:

- Use `dataPort` from the descriptor; do not assume.
- **A descriptor may only strengthen, never silently weaken, a saved security
  setting.** If a bookmark says "use HOPE" and the descriptor claims HOPE is
  unavailable, refuse with a visible error - do not downgrade to plaintext (the
  info port is unauthenticated and MITM-able).
- Treat timeout / connection-refused / EOF / malformed as "no info protocol";
  fall back to your saved transport plan and negative-cache the host for a while.
- The info port is advisory. The real capability contract is still negotiated in
  the login handshake - `capabilities.voice` here is a hint that the *server*
  supports voice, not that *you* will get it.

**[Legacy]** Skip this entirely; connect to the data port with your configured
transport.

### 4.3 TCP handshake (TRTP)

**[Both]** Every data-port session starts with a fixed handshake (base protocol,
*Session Initialization*). The client writes 12 bytes:

| Field | Size | Value |
|---|---|---|
| Protocol ID | 4 | `"TRTP"` (`0x54525450`) |
| Sub-protocol ID | 4 | `"HOTL"` (`0x484F544C`) |
| Version | 2 | `1` |
| Sub-version | 2 | implementation-defined (`0` is fine) |

> **The sub-protocol ID MUST be `"HOTL"`.** Although the original protocol
> describes this field as "user defined", real Hotline clients always send
> `HOTL`, and hardened servers (e.g. Janus) reject any other value as a
> non-Hotline probe - and may **auto-ban the source IP**. Send `HOTL`.

The server replies with 8 bytes:

| Field | Size | Value |
|---|---|---|
| Protocol ID | 4 | `"TRTP"` |
| Error code | 4 | `0` = OK; non-zero = rejected, close the connection |

After a successful handshake the connection carries Hotline transactions.

### 4.4 TLS transport (Modern)

**[Modern]** TLS is the second way (besides HOPE) to encrypt a session, and the
simplest: it is **ordinary TLS wrapping the whole TCP connection** - there is no
Hotline-specific TLS framing. Everything in this guide happens *inside* the TLS
tunnel.

- **Connect to the TLS port** (advertised as `tlsPort` in the info descriptor,
  default 5600 - a *different* port from the plaintext data port), perform a
  standard TLS client handshake, **then** do the TRTP handshake
  ([§4.3](#43-tcp-handshake-trtp)) and everything else over the encrypted socket.
- The **TLS file-transfer port is `tlsPort + 1`** (not `base + 1`). When a
  session is over TLS, open transfer connections ([§14](#14-user-to-user-file-transfer))
  to `tlsPort + 1`, also wrapped in TLS. There is no separate per-transfer
  encryption over TLS - the TLS tunnel already protects the bytes.
- **Certificate verification.** Verify the server certificate against the
  hostname you intend to connect to. Prefer the `Hostname`/`server.hostname` the
  user bookmarked or the info descriptor advertised, not a bare IP, so the
  certificate's SAN can be checked. Servers commonly enforce a **minimum TLS
  version of 1.2** (some require 1.3); negotiate 1.2+ and let the handshake fail
  loudly rather than downgrading.
- **TLS and HOPE are independent and composable.** TLS encrypts the socket; HOPE
  authenticates the password (and, on a *plaintext* socket, can additionally
  encrypt the stream). Inside a TLS tunnel you can still use HOPE for
  authentication but you do **not** need HOPE transport encryption - the TLS
  layer already provides confidentiality, so negotiate HOPE with **no cipher
  fields** (authenticate only). You may use TLS, HOPE-encryption, both, or
  (legacy) neither.

### 4.5 Choosing a transport

Resolve the transport **before** login, using (Modern) the info descriptor and
the user's saved security preference. A reasonable decision order:

1. If the bookmark/descriptor says a transport is **required**
   (`transport.hope.required` or `transport.tls.required`), honour it. If you
   cannot satisfy a required-and-stronger setting, **refuse with a visible
   error** - never silently downgrade ([§4.2](#42-info-port-discovery-modern)).
2. **[Modern], first contact:** prefer the strongest the descriptor advertises -
   TLS (verified) or HOPE-AEAD over plaintext. Use HOPE with `HMAC-SHA256` +
   `CHACHA20-POLY1305` for the encrypted-on-plaintext path.
3. **[Legacy]:** HOPE with a stream cipher (RC4/Blowfish) if the server supports
   it and you can implement it; otherwise plaintext on the base data port.
4. Plaintext is acceptable only when the operator allows it
   (`transport.plaintext.accepted`) and the user has not asked for encryption.

Whatever you pick, the layers above the socket - TRTP handshake, transactions,
messaging flows - are identical. Transport is a property of the byte pipe, not of
the application logic.

---

## 5. Wire framing primitives

Everything after the handshake - login, messaging, file signaling, voice
signaling - is a **transaction**. Implement this layer once and reuse it for
every flow.

### 5.1 Transaction header

Every transaction begins with a 20-byte header:

| Offset | Field | Size | Notes |
|---:|---|---:|---|
| 0 | Flags | 1 | Reserved; send `0` |
| 1 | Is-reply | 1 | `0` = request/notification, `1` = reply |
| 2 | Type | 2 | Transaction type ID (e.g. 810 for IM Send) |
| 4 | ID (task ID) | 4 | Unique per outstanding request; see [§5.4](#54-task-ids-replies-and-notifications) |
| 8 | Error code | 4 | Reply only: `0` = success, non-zero = failure |
| 12 | Total size | 4 | Total payload size across all parts |
| 16 | Data size | 4 | Payload size in *this* part |

`Total size` and `Data size` are equal unless a transaction is split into
multiple parts. **A messaging client can treat them as equal** - messaging
payloads are small (an IM body is capped at `MaxMessageBytes`, default 4096).
If you ever see `Data size < Total size`, accumulate parts until you have
`Total size` bytes of payload; otherwise just read `Data size` bytes.

The payload is a **parameter (field) list**.

### 5.2 Field list and field encoding

The payload starts with a 2-byte field count, then that many fields:

```
u16:fieldCount
repeat fieldCount times:
    u16:fieldID
    u16:fieldSize
    bytes[fieldSize]:fieldData
```

There are three logical field types - **integer**, **string**, **binary** - but
they are all framed identically (ID, size, bytes). The type only tells you how
to interpret `fieldData`:

- **integer**: big-endian unsigned over `fieldSize` bytes (see [§5.3](#53-integer-width-rules)).
- **string**: text in the negotiated encoding (see [§6](#6-text-encoding)).
- **binary**: opaque bytes (GUIDs, relay refs, packed structures).

To **build** a transaction: serialise each field, count them, prepend the field
count, compute payload length, fill the header, send header + payload.

To **parse** one: read 20-byte header, read `Data size` bytes of payload, read
the field count, then loop reading `(id, size, data)`. **Collect fields into a
multimap** - a field ID can legitimately repeat (e.g. several
`DATA_FRIEND_LOGIN` in a roster reply, several `DATA_FRIEND_LOGIN` in a call
invite). Never assume a field appears at most once.

**You MUST ignore unrecognised field IDs and unrecognised transaction types.**
Do not reject a transaction because it carries a field you do not know - this is
what keeps the protocol forward-compatible.

### 5.3 Integer width rules

Two rules coexist; know which applies.

- **Legacy integer fields** (e.g. `FieldUserID` 103, `FieldChatID` 114,
  `FieldVersion` 160) follow the base protocol's *minimal-width* rule: a value
  that fits in 2 bytes is sent as 2 bytes, otherwise 4 bytes. **When reading,
  always honour `fieldSize`** - read the integer over exactly `fieldSize` bytes,
  big-endian. Do not assume 2 or 4.
- **Messaging fields have fixed, declared widths** (spec, *Data Objects*):
  `DATA_PRESENCE_STATE`, `DATA_ROSTER_STATE`, `DATA_ACK_TYPE`,
  `DATA_TYPING_STATE`, `DATA_REASON_CODE`, `DATA_DISCOVERABLE`,
  `DATA_FRIEND_CAPABILITIES` are **`u16` (2 bytes)**;
  `DATA_MESSAGE_TIMESTAMP` is **`u64` (8 bytes)**; `DATA_MESSAGE_GUID` and
  `DATA_FILE_TRANSFER_GUID` are **16 bytes**; `DATA_FILE_RELAY_REF` and
  `DATA_CALL_ID` are **4 bytes**. Emit these at their declared width. When
  reading, still trust `fieldSize`, but expect these widths.

Defensive reader: `readUint(data) = fold bytes big-endian` over the whole field;
it works for both 2- and 4-byte legacy integers and for fixed-width messaging
integers alike.

### 5.4 Task IDs, replies, and notifications

The protocol multiplexes two interaction patterns over one socket. Dispatch
**by the is-reply flag and task ID**, not by guesswork.

- **Request → reply.** You send a transaction with a **unique non-zero task ID**
  and is-reply `0`. The server eventually sends a transaction with **the same
  task ID** and is-reply `1`. Match it to your pending request by task ID. The
  reply's `Error code` (header offset 8) is `0` on success; on failure it is
  non-zero and the payload SHOULD carry `DATA_REASON_CODE` (and may carry
  `FieldError` text). Examples: Get Roster (800), Add Friend (802), IM Send
  (810).
  > **Do not rely on a reply's `Type` field.** A server MAY send replies with
  > `Type` `0`, echoing only the task ID (the Janus reference server does this:
  > its reply carries the request's task ID but a zero type). Route replies
  > **solely by task ID** - keep a `taskID → request type` map at send time if
  > you need to know which request a reply answers (you will, to tell a Login
  > reply from an Add Friend reply). The Login (107) reply, whose `Type` is `0`,
  > is the first one you will hit.
- **Server-initiated notification.** The server sends with **task ID `0`** and
  is-reply `0`, unprompted. There is no transaction-layer reply. Handle these as
  events. Examples: Roster Entry (801), Friend Request (804), Presence Changed
  (809), IM Deliver (811). Application-level acknowledgement, where required, is
  a *separate* transaction (e.g. you answer an IM Deliver with an IM Acknowledge
  812, with its own new task ID).

Implementation: keep a `pendingRequests: taskID → callback/future` map. Allocate
task IDs from a monotonically increasing counter (start at 1; skip 0). On every
inbound transaction:

```
if is_reply == 1:
    future = pendingRequests.remove(header.taskID)
    future.complete(header.errorCode, fields)
else:
    dispatchNotification(header.type, fields)   # task ID is 0
```

Some transactions are **bidirectional** (IM Acknowledge 812, IM Typing 813,
File Offer/Accept/Decline 814–816, Call Invite/Accept/Decline/Cancel 818–821):
the *same type* is a request when you send it and arrives as a notification when
the server forwards a peer's. The is-reply flag and task-ID-0 rule disambiguate
direction unambiguously - always trust them.

### 5.5 Repeated-entry (list) encoding

Hotline transactions carry a **flat** field list, so multi-entry replies (the
roster snapshot, search results) are delimited by a marker field. For messaging,
**`DATA_FRIEND_LOGIN` (0x0600) is the delimiter**:

> Every field following a `DATA_FRIEND_LOGIN`, up to the next
> `DATA_FRIEND_LOGIN` (or end of transaction), belongs to that entry.
> `DATA_FRIEND_LOGIN` is therefore the **first** field of each entry.

So to parse a Get Roster (800) reply or a User Search (823) reply: scan the
field list in order; each time you hit `DATA_FRIEND_LOGIN`, start a new entry and
attach subsequent fields to it until the next `DATA_FRIEND_LOGIN`.

A server MAY instead deliver the roster snapshot as a sequence of individual
Roster Entry (801) notifications after the reply. **Your client MUST accept
either form** - a single reply with repeated entry-groups, or a reply followed
by N separate 801s.

---

## 6. Text encoding

All human-readable string fields are transcoded by the server to each client's
negotiated encoding. Full spec:
[Text Encoding Extension](../Protocol/Capabilities-Text-Encoding.md).

- **[Modern]** Advertise `CAPABILITY_TEXT_ENCODING` (bit 1). If the server
  echoes it, **all** string fields are UTF-8 in both directions. Store the
  negotiated mode as a per-session flag.
- **[Legacy]** Do not advertise bit 1. The server falls back to its default
  encoding - almost always **Mac Roman** (IANA `macintosh`, codepage 10000). A
  legacy client MUST encode outgoing strings to Mac Roman and decode incoming
  strings from Mac Roman.
- Implement one pair of functions and route every string field through them:
  - `decode(wireBytes) → string`: UTF-8 mode returns bytes as-is; otherwise
    decode from Mac Roman.
  - `encode(string) → wireBytes`: UTF-8 mode emits UTF-8; otherwise encode to
    Mac Roman, replacing unmappable characters with `?` (0x3F).
- **Logins SHOULD be printable ASCII** (the safe intersection of all encodings),
  so `DATA_FRIEND_LOGIN` is effectively encoding-independent in practice - but
  still route it through `encode`/`decode` for correctness.
- Binary fields (GUIDs, presence state, timestamps, relay refs, packed member
  lists) are **never** transcoded.
- Line endings: Mac Roman peers use CR (`0x0D`); the server normalises to LF
  (`0x0A`) internally. A legacy client receives CR in message bodies and SHOULD
  emit CR; a UTF-8 client uses LF. Normalise on display as needed.

Messaging string fields you will transcode: `DATA_FRIEND_LOGIN`,
`DATA_FRIEND_NICKNAME`, `DATA_PRESENCE_STATUS_TEXT`, `DATA_MESSAGE_BODY`,
`DATA_REQUEST_NOTE`, `DATA_SEARCH_QUERY`, plus reused `FieldUserName` (102).

---

## 7. Login and capability negotiation

### 7.1 Capability bitmask

Capabilities are negotiated with **`DATA_CAPABILITIES`** (field `0x01F0`, 496),
a big-endian bitmask. Send it in the Login (107) request with the bits you
support; read it back from the login reply to learn what the server **confirmed
for this session**. Absent in the reply = treat as zero = standard mode.

| Bit | Mask | Name | Profile use |
|---:|---|---|---|
| 0 | 0x0001 | `CAPABILITY_LARGE_FILES` | Modern (optional): 64-bit file sizes for large transfers |
| 1 | 0x0002 | `CAPABILITY_TEXT_ENCODING` | Modern: UTF-8 |
| 2 | 0x0004 | `CAPABILITY_VOICE` | Modern: calls |
| 6 | 0x0040 | `CAPABILITY_MESSAGING` | **Both: master switch - required** |
| 7 | 0x0080 | `CAPABILITY_DIRECT_TRANSFER` | Modern: P2P file path |
| 8 | 0x0100 | `CAPABILITY_MESSENGER_SESSION` | **Both: set it - you are a pure messenger** |
| 9 | 0x0200 | `CAPABILITY_MODERN_DATES` | Set it **only** once you *display* modern dates - see the note below. Neither reference messenger sets it |

**Set bit 8.** It declares that this session will never join public chat, ask
for the user list, or accept a classic private message - and a server that
confirms it hides you from the classic Hotline world: you are left out of Get
User Name List (300) replies and your join/leave/away announcements (301/302)
are suppressed. Without it, everyone on the server sees a user in the list who
cannot be chatted with, PM'd, or invited anywhere. You stay fully reachable
through the messaging layer either way; the bit costs you nothing and spares
the file-sharing users a ghost. (A *hybrid* client - a full Hotline client that
also does messaging - must **not** set it.) The server confirms bit 8 only
alongside bit 6.

Typical advertised values:

- **Legacy messenger:** `0x0140` = bits 6,8 (messaging + messenger session), or
  `0x0142` if you can do UTF-8. This is what the Iris reference client sends.
- **Modern messenger:** `0x01C7` = bits 0,1,2,6,7,8 (large files + text
  encoding + voice + messaging + direct transfer + messenger session). This is
  what the Nyx reference client sends, minus voice on a build without an audio
  backend and minus text encoding when forced to Mac Roman.

> **Bit 9 is not a free upgrade - a pure messenger almost certainly wants it
> clear.** `CAPABILITY_MODERN_DATES` selects a *wire format*, and it is the one
> capability a server must never infer or grant on your behalf
> ([spec](../Protocol/Capabilities.md#date-format-selection)). Set it
> and the server stops sending the Mac 1904 encoding; a client that does not
> render the modern form then shows every file and news date in 1904.
>
> **Neither reference messenger sets it**, and the reason is the same for both
> even though they are very different clients: a messenger has no Hotline date
> on screen. This extension's own timestamps are `DATA_MESSAGE_TIMESTAMP` -
> plain Unix seconds, which this bit does not touch ([§8](#8-messaging-object--transaction-reference)).
> Iris decodes no Hotline 8-byte date at all. Nyx decodes both encodings,
> because parsing an FFO `INFO` fork requires it, and then never reads the
> result: outbound it writes the local file's own modification time, and a
> download does not restore timestamps. Nothing a user sees comes from that
> field, so the 1904 encoding's 2040-02-06 ceiling costs nothing.
>
> That is the test to apply to your own client - not "do I touch files?" but
> **"does a server-supplied date reach the screen or the filesystem?"** If it
> does not, leave the bit clear. If it does, advertise it in the same change
> that adds the rendering, never before.

**Gate everything on the reply.** If the server does not echo
`CAPABILITY_MESSAGING` (bit 6), the messaging subsystem is unavailable for this
session - the server has it disabled, or your account lacks the `AccessMessaging`
privilege (bit 58). The client MUST then disable all messaging UI and MUST NOT
send any 800-block transaction. Likewise, only show voice UI if bit 2 came back,
and only attempt the direct file path if bit 7 came back *and* the specific
friend advertises it (see [§14](#14-user-to-user-file-transfer)).

> **Why messaging might not be confirmed even though you asked:** the server
> confirms bit 6 only when (a) `Messaging.Enabled` is true server-side **and**
> (b) your account holds the `AccessMessaging` access privilege. Both are
> server-operator controls. Surface a clear "messaging not available on this
> server / for this account" state rather than failing silently.

### 7.2 Legacy plaintext login

**[Legacy]** The classic Login (107) request carries these fields. **Both** the
login and the password use the legacy **bitwise-inversion** obfuscation: each
byte is one's-complemented (`b → ~b & 0xFF`) before sending.

| Field | ID | Value |
|---|---|---|
| User login | 105 | `inverse(login)` (bitwise-inverted) |
| User password | 106 | `inverse(password)` (bitwise-inverted - see note) |
| Version | 160 | client protocol version (e.g. 151) |
| User name | 102 | display nickname |
| User icon ID | 104 | an icon number (any small integer; messengers can send 0) |
| Capabilities | 0x01F0 | your capability bitmask |

> **Invert the password. Do not send it in the clear.** This is the one place
> where getting it wrong produces a login that simply never succeeds, with an
> "Incorrect login." that tells you nothing about why.
>
> The reasoning is worth following, because a modern server looks like it should
> want plaintext. Janus stores a **bcrypt** hash and cannot reverse it, so it
> does not de-obfuscate anything: it takes the `FieldUserPassword` bytes exactly
> as they arrive off the wire and bcrypt-compares them. But the hash it compares
> against was computed over the *inverted* bytes - `bcrypt(inverse(password))` -
> because that is the form every classic client and admin tool has always sent.
> So the wire bytes must be inverted for the comparison to land, even though the
> server never inverts anything itself.
>
> The inversion is not security - it is trivially reversible, and it was never
> anything more than obfuscation. It is a *wire format*. For real
> confidentiality over a plaintext socket use HOPE or TLS; under HOPE the
> password is MAC'd instead and the real plaintext keys the MAC, per
> [§7.3](#73-hope-secure-login).

Send with a fresh task ID, is-reply 0. The server replies (is-reply 1, same
task ID) - see [§7.4](#74-reading-the-login-reply).

### 7.3 HOPE secure login

**[Both, recommended]** HOPE adds real password authentication and optional
transport encryption, piggybacking on Login (107) - no new transaction types.
Full spec: [HOPE Secure Login](../Protocol/HOPE-Secure-Login.md). It
is a **three-step** exchange.

**Step 1 - Identification.** Send Login (107) with:

| Field | ID | Value |
|---|---|---|
| User login | 105 | a single null byte `0x00` (signals HOPE identification) |
| User password | 106 | a single null byte `0x00` (placeholder) |
| MAC algorithm | 0x0E04 | algorithm list, preferred first (see encoding below) |
| App ID | 0x0E01 | your 4-byte app id (e.g. `"KLNC"`); MAY be omitted |
| App string | 0x0E02 | `"MyMessenger 1.0"`; MAY be omitted |
| Session key | 0x0E03 | empty (requests a key) |
| Client cipher | 0x0EC2 | **[Modern, optional]** cipher list for client→server |
| Client compression | 0x0ECA | **[optional]** compression list |

Algorithm-list encoding (used for MAC, cipher, and compression lists):

```
u16:count  [ u8:nameLength  bytes:name ]*
```

MAC algorithms, strongest to weakest: `HMAC-SHA256`, `HMAC-SHA1`, `SHA1`,
`HMAC-MD5`, `MD5`, `INVERSE`. Both sides MUST support `INVERSE` as the fallback.
A modern client offers `["HMAC-SHA256","HMAC-SHA1","INVERSE"]`.

> **Compression gotcha:** never send an *empty* compression list (count=0) - it
> crashes some classic servers/clients. Omit the field entirely if you are not
> negotiating compression.

**Step 2 - Identification reply.** The server replies with a task transaction
(note: its header `Type` is set so the first 4 header bytes read as
`0x00010000`, a HOPE quirk for classic-client compatibility - your parser keys
off the is-reply flag and task ID, so just read the fields). It contains:

| Field | ID | Meaning |
|---|---|---|
| Session key | 0x0E03 | **64 bytes**: `serverIP[4] serverPort[2] random[58]` |
| MAC algorithm | 0x0E04 | the single chosen algorithm (`u16:1  u8:len  name`) |
| User login | 105 | **empty** ⇒ do not MAC the login (invert it); **non-empty** (an algorithm name) ⇒ MAC the login too |
| Server app id/string | 0x0E01 / 0x0E02 | informational |
| Cipher/mode/IV/compression | 0x0EC1…0x0ECA | **[optional]** the negotiated transport parameters |

**Validate the session key's embedded address.** The first 6 bytes are the
server's IP:port from its side of the socket. Compare against the address you
connected to; on mismatch, warn or disconnect (MITM/NAT indicator), matching
classic `hx` behaviour.

Compare against the port you actually **dialled**, which inside a TLS tunnel
([§4.4](#44-tls-transport-modern)) is `tlsPort`, not the base data port. Both
sides get this wrong in the same way: a server that fills the field from its
configuration rather than from the accepted socket reports the base port to
every TLS client, and a client that compares against the base port rather than
the one it opened flags every TLS session. Either mistake produces a
man-in-the-middle warning on a perfectly ordinary login - and a check that
cries wolf on every connection is worse than no check, because users learn to
click through it.

**Make sure the warning can actually be seen.** It is raised mid-handshake,
before your main window exists, and a sign-on screen that has no place to show
it will drop it silently. Hold warnings raised during login and present them
once there is a window, or you will ship a security check whose output goes
nowhere.

**Step 3 - Authenticated login.** Send Login (107) again with real credentials:

| Field | ID | Value |
|---|---|---|
| User login | 105 | `inverse(login)` if the server's login field in Step 2 was empty; else `mac(login, sessionKey)` |
| User password | 106 | `mac(password, sessionKey)` using the chosen algorithm |
| User name | 102 | display nickname |
| User icon ID | 104 | icon number |
| Capabilities | 0x01F0 | your capability bitmask |
| Server cipher / compression echo | 0x0EC1 / 0x0EC9 | **[optional]** echo the server's Step-2 selections to confirm |

MAC computation (HMAC variants): `mac = HMAC(key = passwordBytes, msg =
sessionKey)`. The plain `SHA1`/`MD5` variants are `hash(password ‖ sessionKey)`.
`passwordBytes` is the **real, un-inverted** UTF-8 password - the opposite of
the legacy field in [§7.2](#72-legacy-plaintext-login). Inversion is a property
of that one wire field, not of the credential; every HOPE MAC and every key
derived from one uses the plaintext. The sole exception is the `INVERSE`
fallback algorithm, which *is* the legacy inversion wearing a MAC's name.

**Transport encryption [Modern].** If a cipher was negotiated, derive keys after
the password MAC:

```
encode_key = MAC(key = passwordBytes, msg = password_mac)
decode_key = MAC(key = passwordBytes, msg = encode_key)
```

The client encrypts outbound with `decode_key` and decrypts inbound with
`encode_key` (the server uses them the other way around). From the end of Step 3
onward, **every** transaction (header + payload) is encrypted at the byte-stream
level.

- **Stream ciphers (RC4, Blowfish-OFB)** - legacy-compatible; support key
  rotation signalled in the top 8 bits of the encrypted header's type field.
  Mirror the sender's encrypt-header → encrypt-first-2-body-bytes → rotate →
  encrypt-rest sequence exactly to stay in sync.
- **ChaCha20-Poly1305 (AEAD) [Modern, preferred]** - uses framed AEAD instead of
  the stream wire format; key derivation, frame structure, nonce construction,
  and transfer-port encryption are specified separately in
  [HOPE ChaCha20-Poly1305](../Protocol/HOPE-ChaCha20-Poly1305.md).
- Optional compression (GZIP/LZ4/ZSTD) is a persistent streaming codec applied
  *before* encryption outbound, *after* decryption inbound.

If you do not need transport encryption (e.g. you are inside a TLS tunnel, or
this is the legacy plaintext profile), simply omit all cipher/compression fields
in Step 1 and HOPE authenticates the password without encrypting the stream.

### 7.4 Reading the login reply

The successful Login (107) reply (is-reply 1, error code 0) carries:

| Field | ID | Use |
|---|---|---|
| Version | 160 | server protocol version |
| Server name | 162 (0x00A2) | display |
| User ID | 103 | **your own** session's 16-bit user ID for this connection |
| Capabilities | 0x01F0 | **the confirmed session capabilities - gate your UI on this** |
| Banner id | 161 | ignorable for a pure messenger |
| `DATA_MAX_MESSAGE_BYTES` | 0x0620 | `u32` - longest IM body this server accepts |
| `DATA_MAX_ROSTER_SIZE` | 0x0621 | `u32` - most roster entries this account may hold |
| `DATA_MAX_OFFLINE_QUEUE` | 0x0622 | `u32` - deepest offline queue per recipient |

A non-zero error code means login failed (bad credentials, account locked,
version floor not met, etc.); the payload carries `FieldError` (100) text to
show the user.

**The three limit fields are the only chance you get to learn them.** Every one
of these caps is operator-configurable, and without the advertisement the sole
way to discover a non-default value is to have a request refused - *after* the
user wrote the message or added the friend, when the only remedy is to undo it.
Capture them at login and pre-validate against them.

| Rule | |
|---|---|
| Field absent | Use the default: 4096 bytes, 500 entries, 500 queued. A server predating this feature sends none of the three. |
| Field present, value `0` | Not meaningful for these three - use the default. `0` means "unlimited" only for `MaxSessionsPerLogin`, which is not advertised. |
| Sent only when messaging is confirmed | If bit 6 was not echoed, they will not appear. |

They are advisory only in the sense that the server enforces them on every
request regardless, so there is nothing to gain by ignoring them; treat them as
authoritative for your own pre-checks. **`MaxMessageBytes` bounds the encoded
body, not the character count** - the same sentence costs more in UTF-8 than in
Mac Roman, so measure the bytes you are about to send, and show remaining
capacity rather than an error after the fact.

### 7.5 Post-login: the pure-messenger shortcut

A classic Hotline client, after login, shows an agreement, fetches the user
list, file list, and news. **A pure messenger does almost none of that.** The
minimal, conforming post-login sequence is:

1. If the server sent **Show Agreement (109)** (a notification with the
   agreement text in `FieldData` (101)), present it and reply with **Agreed
   (121)** carrying your `FieldUserName` (102), `FieldUserIconID` (104), and an
   options field (113) - exactly as a normal client would. Some servers gate the
   session behind agreement acceptance. If `No server agreement` (field 154 = 1)
   was set, skip.
2. **Do not** send Get User Name List (300), Get File Name List (200), or any
   news transaction unless you actually want those features. A pure messenger
   ignores public chat and files-as-server-content.
3. Send **Set Presence (808)** with your state, optional status text, and your
   discovery preference - **before** the roster sync, not after.
4. Send **Get Roster (800)** (see [§9](#9-roster-and-friend-management)). This is
   the "client is ready" signal: the server uses it to deliver your roster,
   pending friend requests, **and your offline message backlog**. A client that
   never sends Get Roster will not receive queued messages.

From here the client is live: handle inbound notifications (801, 804, 809, 811,
812, 813, forwarded 814–821) and issue requests in response to user actions.

> **Why presence goes first.** A Login is online when it has a live session
> ([spec, *Server Routing*](../Protocol/Capabilities-Messaging.md#server-routing)),
> so the server announces you to your friends as soon as your session
> registers - before you have said anything about what state you are in. Until
> your Set Presence lands, everyone is looking at that default.
>
> On a first sign-on the default is usually right. On a **reconnect** it
> frequently is not: someone whose connection dropped while Away or Busy comes
> back announced as `Online`, which in most clients means a "buddy just signed
> on" alert for every friend they have. Sending presence first shrinks that
> window to one round trip; sending it after Get Roster stretches it across the
> roster reply *and* the offline backlog that rides along with it - which is
> longest exactly when the user has been away the longest.
>
> Nothing depends on the order: the backlog drains on Get Roster whenever it
> arrives, and the roster snapshot carries your friends' presence regardless of
> what you have told the server about your own.

### 7.6 ChaCha20-Poly1305 AEAD wire format (Modern)

**[Modern]** This is the recommended HOPE transport encryption - confidentiality
and integrity in one primitive, no key-rotation bookkeeping. Negotiate it in the
HOPE handshake ([§7.3](#73-hope-secure-login)); this section is the complete wire
contract. Normative source:
[HOPE ChaCha20-Poly1305](../Protocol/HOPE-ChaCha20-Poly1305.md).

If you are running inside a **TLS tunnel** you do **not** need this - TLS already
encrypts the stream, so authenticate with HOPE and skip the cipher fields. Use
AEAD when you want encryption over an otherwise-plaintext socket.

**Negotiation.** Offer `CHACHA20-POLY1305` in your client cipher list (0x0EC2);
the server confirms it in 0x0EC1/0x0EC2 and signals AEAD via the mode fields
(0x0EC3/0x0EC4 = `"AEAD"`) and checksum fields (0x0EC7/0x0EC8 = `"AEAD"`, meaning
"integrity is in the AEAD tag, no separate checksum"). You MAY also infer AEAD
from the cipher name alone. **`INVERSE` cannot be used with AEAD** (HKDF needs
real key material) - offer a real MAC (`HMAC-SHA256` recommended; its 32-byte
output natively matches the key size).

**Key derivation.** Start from the standard HOPE MAC keys, then HKDF-expand each
to 32 bytes (HKDF-SHA256, RFC 5869):

```
password_mac   = MAC(key=passwordBytes, msg=sessionKey)
encode_key     = MAC(key=passwordBytes, msg=password_mac)
decode_key     = MAC(key=passwordBytes, msg=encode_key)

encode_key_256 = HKDF-SHA256(ikm=encode_key, salt=sessionKey, info="hope-chacha-encode")
decode_key_256 = HKDF-SHA256(ikm=decode_key, salt=sessionKey, info="hope-chacha-decode")
```

`MAC` is the negotiated algorithm (MD5/SHA1/HMAC-MD5/HMAC-SHA1/HMAC-SHA256 - not
INVERSE). As with stream mode, the **client writes with `decode_key_256` and
reads with `encode_key_256`** (the server uses them the other way around).

**Frame structure.** The XOR-stream wire format is replaced by length-prefixed
AEAD frames. Each whole Hotline transaction (20-byte header + field-list body) is
sealed as **one** frame:

```
+-------------------+------------------------------------+
| Length (u32, BE)  | Ciphertext + 16-byte Poly1305 tag  |
+-------------------+------------------------------------+
```

- `Length` = byte length of `Ciphertext + Tag` (i.e. plaintext length + 16). It
  is **not** itself encrypted or authenticated.
- `Ciphertext + Tag` = `ChaCha20-Poly1305.Seal(plaintext)` where `plaintext` is
  the exact bytes the transaction would have on an unencrypted wire.
- **No compression** is used in AEAD mode.

To read: read 4 bytes → `Length`; read `Length` bytes; `Open` them with the
receive key and the next receive nonce; parse the result as a normal transaction.

**Nonce construction.** 12-byte (96-bit) nonce, deterministic, per-direction:

```
byte 0      : direction  (0x00 = server→client, 0x01 = client→server)
bytes 1–3   : 0x00 0x00 0x00
bytes 4–11  : counter (u64, big-endian), starts at 0, +1 per frame sent in that direction
```

Each side keeps **separate send and receive counters**. The direction byte
guarantees the two endpoints never reuse a nonce under the shared key. Enforce a
**maximum frame size of 16 MiB** on what you *read*, to bound memory against a
hostile length prefix.

**Cap what you write far lower.** The 16 MiB figure is a memory-safety ceiling
on what you must tolerate, not a target for what you emit. Chunk outbound writes
at **256 KB or below**, for two reasons that outlive any particular server:

- **A peer may enforce a lower ceiling than the specification names.** The
  figure is a `should`, and an implementation is free to be stricter - the Janus
  reference server enforced 256 KB for exactly this stretch of its life. A
  sender that stays well under the limit interoperates with strict and permissive
  peers alike; one that emits 16 MiB frames is betting on every peer having read
  the same sentence the same way.
- **A whole-fork frame defeats streaming.** A file transfer reuses this exact
  frame format ([§14.4](#144-transfer-port-encryption)), and an implementation
  that seals an entire DATA fork as one frame forces both ends to hold the whole
  thing in memory before a byte can be decrypted - and gives the receiver no
  progress to report. Chunking is what makes a transfer resumable-feeling and
  measurable, quite apart from any limit.

The reference client chunks, which is why its transfer code and its control code
share one framing layer.

This same frame format, nonce rule, and 16 MiB cap also govern the encrypted
**file-transfer** connections - see [§14.4](#144-transfer-port-encryption).

---

## 8. Messaging object & transaction reference

The complete normative tables are in the spec (*Transaction Types*, *Data
Objects*, *Enumerations*); Appendices [B](#appendix-b--field-id-quick-table) and
[C](#appendix-c--transaction-id-quick-table) reproduce the IDs for quick lookup.
Two reused field families you will also touch:

- **File metadata** in offers: `FieldFileName` (201, string),
  `FieldFileSize` (207, integer) or `DATA_FILESIZE64` (0x01F1, 8-byte, large
  files), `FieldFileTypeString` (205), `FieldFileCreatorString` (206).
- **Voice** during a call: `DATA_VOICE_SDP` (0x01F5), `DATA_VOICE_ICE`
  (0x01F6), `DATA_VOICE_PARTICIPANTS` (0x01F9), etc. - see
  [§15](#15-call-signaling-and-voice-modern).

All messaging transaction IDs are in the **800 block**; all messaging field IDs
are in the **0x0600 block**. Timestamps in this extension are **`u64` Unix
seconds** (`DATA_MESSAGE_TIMESTAMP`), *not* the legacy Hotline date structure,
so there is no 1904-epoch ambiguity to resolve and `CAPABILITY_MODERN_DATES`
(bit 9) does not apply to them. A messaging client that also browses files or
news still has to decide about that bit; see
[DATA_CAPABILITIES](../Protocol/Capabilities.md#date-format-selection).

---

## 9. Roster and friend management

### 9.1 Roster sync on login - Get Roster (800)

Request/reply, no request fields. Send it after login. The reply (and/or the
801 notifications that may follow) enumerates your roster. Each entry is a group
of fields beginning with `DATA_FRIEND_LOGIN` (see
[§5.5](#55-repeated-entry-list-encoding)):

| Field | ID | Presence |
|---|---|---|
| `DATA_FRIEND_LOGIN` | 0x0600 | REQUIRED, first in each entry |
| `DATA_FRIEND_NICKNAME` | 0x0601 | optional (your private alias) |
| `DATA_ROSTER_STATE` | 0x0604 | the relationship state ([Appendix A](#appendix-a--enumerations)) |
| `DATA_PRESENCE_STATE` | 0x0602 | `Accepted` entries only, and only when the friend is visible |
| `DATA_PRESENCE_STATUS_TEXT` | 0x0603 | `Accepted` entries only; present whenever the friend has a status, **even if they are offline** |
| `DATA_FRIEND_CAPABILITIES` | 0x0613 | `Accepted` entries only, and only when the friend is online |
| `FieldUserName` | 102 | `Accepted` entries only; **the name that friend goes by** - see below |

**Everything past the state is `Accepted`-only.** A `PendingOut`, `PendingIn` or
`Blocked` row arrives as login + state and nothing else - no name, no presence,
no status, no capabilities. Design the row for it: a pending request renders
from the bare Login, and so does someone you have blocked. Do not treat a
missing name on those rows as a server that forgot to send one.

**Within an accepted entry, two of the four survive the friend being offline
and two do not.** Presence and capabilities describe a live session, so they
are absent for a friend who is signed out or invisible. The status text and
`FieldUserName` are stored against the *account*, so they arrive whether or not
that friend is present - and the roster snapshot is the **only** place a client
learns them for a friend whose presence has not changed since it signed in.
Store both on the entry; see the next section for why dropping the status here
is the specific mistake that makes friends' statuses look like they never load.

Build your buddy list from this. **`DATA_FRIEND_CAPABILITIES` is how you gate
per-friend UI**: it is a `u16` carrying that friend's `DATA_CAPABILITIES` bits,
so you can grey out "Call" for a friend without bit 2 and "Send file (direct)"
for one without bit 7.

**`FieldUserName` (102) is a reused base-protocol field, and it is inside the
entry group.** Two things follow, and both have bitten implementations:

- Attribute it to the `DATA_FRIEND_LOGIN` that opened its group. A parser that
  reaches for "field 102 in this transaction" gets one friend's name and paints
  it onto every row in the list.
- **Store it on the roster entry, not on a live-presence object.** The server
  prefers the name the account *published* over whatever a live session calls
  itself, precisely so the value outlives the session. Keep it and your offline
  friends still have names; throw it away when they sign out and your buddy list
  degrades to account names for everyone who is not currently on.

**Display name precedence** - show, in order:

1. `DATA_FRIEND_NICKNAME` - your private alias for them, if you set one.
2. `FieldUserName` - the name they go by.
3. `DATA_FRIEND_LOGIN` - the bare Login, as a last resort.

Do not append the Login to either of the first two. It is an identifier, not
part of anybody's name.

Any `PendingIn` entries (someone wants to add *you*) also arrive as separate
**Friend Request (804)** notifications, including ones queued while you were
offline. Surface these as actionable "X wants to add you" prompts.

### 9.2 Live deltas - Roster Entry (801)

A server-initiated notification carrying one change: an addition, a
relationship-state change, a presence change, or a removal.

- **Removal** is encoded as `DATA_ROSTER_STATE = 0` (`Removed`) with only
  `DATA_FRIEND_LOGIN`. Drop that buddy from the list.
- Every other delta carries the entry's **current** state and the fields that go
  with it - not just the ones that changed. Store it in place of what you had.

Required field: `DATA_FRIEND_LOGIN`, `DATA_ROSTER_STATE`. Optional: nickname,
presence, status text, capabilities, `FieldUserName` (102) - an 801 entry group
is built the same way a snapshot entry is, so the `Accepted`-only rules from
[§9.1](#91-roster-sync-on-login--get-roster-800) apply here too.

**Replace the entry; do not merge into it.** The group in an 801 is everything
the server currently holds for that friend, so store it wholesale - "create if
absent, otherwise overwrite". An absent optional field means *no such value*,
not *unchanged*. The word "delta" is about which entry changed, not which
fields.

This matters most for the fields a user can clear. Clearing your alias for
someone with [Set Friend Nickname (824)](#96-set-friend-nickname-824) produces
an 801 with **no** `DATA_FRIEND_NICKNAME` in it - there is no present-but-empty
form here, and the flat field list has no way to mark a field "unchanged". A
client that merges therefore keeps showing the old alias for the rest of the
session, and nothing later in the protocol will ever dislodge it. The same trap
applies to a cleared status text.

Apply deltas **idempotently** - you will receive 801s for changes you made
yourself (see [§9.5](#95-remove-803-block-806-unblock-807)) - but idempotent
means "applying it twice is the same as once", which overwriting already is. It
is not an argument for merging.

Because a snapshot entry and a delta for the same friend are byte-identical, one
parse-and-store routine serves both [§9.1](#91-roster-sync-on-login--get-roster-800)
and this section. Writing two is how they drift apart.

### 9.3 Add Friend (802)

Request/reply. Mutual-consent (ICQ-style authorization).

Request: `DATA_FRIEND_LOGIN` (REQUIRED, the target), `DATA_REQUEST_NOTE`
(optional, e.g. "Hi, it's Bob from work").

The server validates the target exists and holds `AccessMessaging`. On success
it creates your `PendingOut` row and the target's `PendingIn` row, persists
both, and delivers a Friend Request (804) to the target (now if online, else at
their next login). Reply: `DATA_ROSTER_STATE = PendingOut` on success, else
`DATA_REASON_CODE` with one of:

- `AccountNotFound` (1) - no such Login. **Also returned when the target has
  blocked you** (the server hides block state - see
  [§19](#19-security-considerations-for-client-authors)).
- `NotMessagingEnabled` (2) - the Login exists but cannot use messaging.
- `AlreadyFriends` (4), `RequestPending` (5), `RosterFull` (12),
  `RateLimited` (10).

Show the friend as pending until a Roster Entry (801) flips them to `Accepted`.

### 9.4 Friend Request (804) and Friend Response (805)

**Friend Request (804)** - inbound notification: someone added you. Fields:
`DATA_FRIEND_LOGIN` (the requester), `DATA_FRIEND_CAPABILITIES` (optional),
`DATA_REQUEST_NOTE` (optional). Present accept/reject UI.

**Friend Response (805)** - request/reply you send to answer:
- Accept: `DATA_FRIEND_LOGIN` + `DATA_ROSTER_STATE = Accepted` (3). Both sides
  go `Accepted` and presence starts flowing.
- Reject: `DATA_FRIEND_LOGIN` with `DATA_ROSTER_STATE` absent or `Blocked`. Both
  pending rows are dropped.

Reply: `DATA_REASON_CODE`. **Both** parties' sessions get a Roster Entry (801)
reflecting the outcome - the requester's *and* yours, including the session that
answered. Accepting is not something you have to apply to your own list by hand.

### 9.5 Remove (803), Block (806), Unblock (807)

- **Remove Friend (803)** - request/reply, `DATA_FRIEND_LOGIN`. Withdraws a
  `PendingOut`, rejects a `PendingIn`, or removes an `Accepted` friend.
  **Removal is mutual**: the relationship is deleted in both directions, and the
  server sends a Roster Entry (801) with `DATA_ROSTER_STATE = Removed` to every
  session of *both* parties - **including the session that asked**. Drop the row
  when that 801 arrives; you may also update optimistically, but you must handle
  the echo either way.
- **Block User (806)** - request/reply, `DATA_FRIEND_LOGIN`. Sets `Blocked`. A
  blocked Login cannot message, offer files, call, or friend-request you, and
  cannot see your presence; it is hidden from your discovery results. Blocking an
  existing friend also removes the friendship. Like a removal this touches two
  rosters: your sessions get an 801 with state `Blocked`, and **the blocked
  person's sessions get an 801 with state `Removed`** - they learn the
  relationship ended, never that it ended in a block. On the receiving side you
  cannot tell a block from an ordinary removal, and you are not meant to. Your
  alias for them **survives** the block and arrives in that 801, so a blocked
  row still reads as "Bobby" rather than `bob`.
- **Unblock User (807)** - request/reply, `DATA_FRIEND_LOGIN`. Clears `Blocked`,
  leaving no row (you are strangers again). Your sessions get a `Removed` 801;
  the other party hears nothing, having never been told about the block. Note
  the asymmetry: blocking keeps your alias, unblocking discards it along with
  the row. Don't present the pair as a reversible toggle.

All three reply with `DATA_REASON_CODE`.

> **You are told about your own changes.** Every roster-changing transaction -
> 803, 805, 806, 807, 824 - produces an 801 to your *own* sessions as well as
> the other party's. This is deliberate: a client that waits to be told, rather
> than mutating its list optimistically, would otherwise see an accepted friend
> never appear and a removed friend never go away until its next sign-in. Apply
> incoming 801s idempotently and you can pick either style safely.

### 9.6 Set Friend Nickname (824)

Request/reply. Sets your **private** local alias for a friend (never shown to
them). Fields: `DATA_FRIEND_LOGIN` (REQUIRED), `DATA_FRIEND_NICKNAME`
(REQUIRED; empty clears the alias). Reply: `DATA_REASON_CODE`.

The friend must be `Accepted` - against a pending or blocked row you get
`NotFriends` (6), so gate the "rename" menu item on state. The server fans the
change to **all** your sessions via Roster Entry (801), the one that asked
included, exactly like the other roster-changing transactions below.

---

## 10. User discovery and profiles

Two discovery mechanisms, split by privacy posture, plus the profile an account
publishes to its friends. All three SHOULD be assumed rate-limited; handle
`RateLimited` (10) gracefully (back off, don't hammer).

> **None of the three tells you who is online.** In particular
> `DATA_FRIEND_CAPABILITIES` is **absent from every result for someone who is
> not already your accepted friend** - on Find User, User Search and Get User
> Info alike. Capabilities are negotiated per session, so a server that reported
> them would be reporting that the account is signed in, which is exactly what
> discovery refuses to answer. Do not read the absence as "offline" either: it
> is absent for every non-friend regardless. If you want to know whether someone
> can take a call, add them as a friend and read it off the roster.

### 10.1 Find User (822)

Request/reply. Resolve an **exact** Login (you already know the handle - the
AIM/ICQ model). Always available regardless of the target's discovery
preference. Request: `DATA_FRIEND_LOGIN`. Reply on success:
`DATA_FRIEND_LOGIN`, `FieldUserName` (102, display name), and
`DATA_FRIEND_CAPABILITIES` **only if they are already your friend**. **Does
not** reveal presence/online state for a non-friend. On failure:
`DATA_REASON_CODE` (`AccountNotFound` / `NotMessagingEnabled`). Use this to
validate a handle before sending a friend request, so you can warn "no such
user / can't message that account" up front.

### 10.2 User Search (823)

Request/reply. Directory search over accounts whose own `DATA_DISCOVERABLE`
preference is `1`. Request: `DATA_SEARCH_QUERY` (matches Login and/or display
name). The reply is a **repeated** list (delimited by `DATA_FRIEND_LOGIN`, per
[§5.5](#55-repeated-entry-list-encoding)) of `DATA_FRIEND_LOGIN` +
`FieldUserName` (optional) + `DATA_FRIEND_CAPABILITIES` (friends only, per the
note above), preceded by `DATA_REASON_CODE` = `OK` - start parsing entries at the first
`DATA_FRIEND_LOGIN` and ignore anything before it. Unlisted accounts and
either-direction blocks never appear.

**The result count is bounded and the bound is not advertised**, so a truncated
list looks exactly like a complete one on the wire (the reference server stops
at 50). Never label the results "all users matching X" - say "matches" and offer
to narrow the search. Results carry no profile detail whatsoever: finding
someone is not being allowed to read about them.

### 10.3 User profiles (825 / 826)

An account MAY publish a small profile. **Every field is optional**, and the
Login is not part of it - that is the permanent identity and cannot change.

| Field | ID | Type |
|---|---|---|
| `DATA_PROFILE_NICKNAME` | 0x0614 | string - the name they go by |
| `DATA_PROFILE_FIRST_NAME` | 0x0615 | string |
| `DATA_PROFILE_LAST_NAME` | 0x0616 | string |
| `DATA_PROFILE_EMAIL` | 0x0617 | string |
| `DATA_PROFILE_GENDER` | 0x0618 | `u16` - `0` unspecified, `1` female, `2` male, `3` other |
| `DATA_PROFILE_BIRTHDATE` | 0x0619 | `bytes[4]` - `u16:year, u8:month, u8:day` |
| `DATA_PROFILE_COUNTRY` | 0x061A | string - ISO 3166-1 alpha-2 |
| `DATA_PROFILE_POSTCODE` | 0x061B | string |
| `DATA_PROFILE_LANGUAGE` | 0x061C | string - ISO 639-1, **repeated**, at most 3 |

**Get User Info (825)** - request/reply. Request: `DATA_FRIEND_LOGIN`. The reply
always carries the public card (`DATA_FRIEND_LOGIN`, `FieldUserName`);
`DATA_FRIEND_CAPABILITIES` and the `DATA_PROFILE_*` fields are added **only when
you are accepted friends** (or you are asking about yourself). Otherwise you get
`DATA_REASON_CODE` = `NotFriends` (6) on an *otherwise successful* reply - that
is not an error, it is the server telling you why the panel is empty so you can
offer "add as friend" instead of showing a blank card.

**Set User Info (826)** - request/reply, **replaces** your whole profile. There
is no field naming whose profile to write; the subject is always your
authenticated Login, so one account can never write another's. An absent or
empty field clears that entry, which means you must send every field you want to
keep on every save - read-modify-write, not patch. Reply: `DATA_REASON_CODE`.

Three details worth building for:

- **Birth-date parts are independently optional.** Any of year, month or day may
  be `0` for "not given" - publishing a day and month but not a year is an
  ordinary thing to want. Offer three separate inputs, not one date picker.
- **Servers truncate rather than refuse** over-long strings (the reference
  server bounds each at 128 bytes), so what you wrote is not necessarily what is
  stored. Re-read with 825 after saving instead of trusting your local copy.
- **An unrecognised gender value is stored as unspecified, not rejected**, so a
  newer client is never refused by an older server.

Changing your profile changes the name your friends see, so the server announces
it to them as a **Presence Changed (809)** carrying the new `FieldUserName`
([§11.2](#112-presence-changed-809)).

**Searchability.** [User Search (823)](#102-user-search-823) additionally matches
a discoverable account's nickname, first name and last name on substring. Gender,
birth date, country, postcode and languages are **not** searchable, and e-mail
matches **in full only** - a substring match would let anyone walk the directory
and confirm addresses a fragment at a time.

---

## 11. Presence

### 11.1 Set Presence (808)

Request/reply. Set **your own** state, optional status text, and optionally your
discovery preference. Request fields:

| Field | ID | Notes |
|---|---|---|
| `DATA_PRESENCE_STATE` | 0x0602 | REQUIRED; `Online`/`Away`/`Invisible`/`Busy` (1–4). **Never send `Offline` (0)** - that value is server→client only. |
| `DATA_PRESENCE_STATUS_TEXT` | 0x0603 | optional custom text - see the clearing rule below |
| `DATA_DISCOVERABLE` | 0x0611 | optional; `0` = unlisted (default), `1` = directory-searchable |

Reply: `DATA_REASON_CODE`. The server persists your status text and discovery
preference (restored next login) and fans **Presence Changed (809)** to your
accepted friends - unless you are `Invisible`, which suppresses outbound
presence while you still receive everyone else's.

**Clearing the status text: send the field empty, never omit it.** This is the
one field in the extension where "absent" and "empty" mean different things:

| What you send | What the server does |
|---|---|
| `DATA_PRESENCE_STATUS_TEXT` absent | Keeps whatever text it already has |
| `DATA_PRESENCE_STATUS_TEXT` present, empty (or whitespace) | Clears the text |

A client that omits the field when the user's status box is blank can never
clear a status: the clear reads as "no opinion", the old text is retained, and
it goes on being published to the user's friends. The simplest correct
implementation always sends the field, empty when there is nothing to say.

### 11.2 Presence Changed (809)

Server-initiated notification when an accepted friend's presence changes
(including going online/offline). Fields: `DATA_FRIEND_LOGIN` (REQUIRED),
`DATA_PRESENCE_STATE` (REQUIRED), `DATA_PRESENCE_STATUS_TEXT` (optional),
`DATA_FRIEND_CAPABILITIES` (optional), `FieldUserName` (102, optional).

809 is also how a friend's **name change** reaches you: when someone edits their
profile ([§10.3](#103-user-profiles-825--826)) the server announces it as a
presence change carrying the new `FieldUserName`. So apply 809 to your stored
roster entry - if you only feed it to an online/offline indicator, friends'
names silently go stale until your next full roster sync.

Unlike an 801, an 809 **is** a partial update: it carries presence and whatever
account state came with it, not a whole roster entry, so it updates the fields
it names and leaves the relationship state and your alias alone. Assign the
fields it carries - including a `DATA_PRESENCE_STATUS_TEXT` that is absent,
which is how a cleared status reaches you.

Presence is **aggregated** across the friend's sessions (most-available wins:
`Online` > `Busy` > `Away`). A friend who disconnected **or** who is `Invisible`
is reported identically as `Offline` (0) - the two are indistinguishable by
design; do not try to tell them apart.

#### A friend's status text arrives by two routes, and you need both

809 tells you about a status *change*. It fires when the owner sets it, and it
reaches whoever is signed in at that moment - nobody else. If Alice sets a
status while you are offline and then leaves it alone, no 809 will ever carry it
to you: there is nothing left to change.

The other route is the [Get Roster (800)](#91-roster-sync-on-login--get-roster-800)
snapshot, which carries the current text for every accepted friend. That is what
closes the gap, and it is why the status text is worth storing on the roster
entry rather than treating it as something that rides along with a presence
update. A client that reads `DATA_PRESENCE_STATUS_TEXT` only out of 809 shows
statuses for friends who happened to change one while it was connected, and
blanks for everyone else - which looks like the feature loading unreliably
rather than a client that never asked.

The same reasoning covers the display name, for the same reason: both are
account state, and the snapshot is where account state arrives.

---

## 12. Instant messaging

### 12.1 IM Send (810)

Request/reply. Send a 1:1 message to an `Accepted` friend.

Request: `DATA_FRIEND_LOGIN` (recipient), `DATA_MESSAGE_GUID` (16 bytes,
client-generated - **UUIDv4 RECOMMENDED**), `DATA_MESSAGE_BODY` (text, ≤ the
`DATA_MAX_MESSAGE_BYTES` the server advertised at login, default 4096 -
**encoded bytes, not characters**; see [§7.4](#74-reading-the-login-reply)).

Reply `DATA_REASON_CODE`:
- `OK` (0) - delivered to at least one live session.
- `OfflineQueued` (7) - recipient offline; stored for later (informational
  success - show a "will deliver when online" hint, not an error).
- Errors: `NotFriends` (6), `Blocked` (3), `RateLimited` (10), `QueueFull` (9),
  `MessageTooLong` (13). Older servers reject an over-long body with a failure
  and no reason code at all, so decide from the header's error code and use the
  reason code only to word the message you show.

**Dedup is by GUID.** Re-sending the same `(GUID, recipient)` is idempotent - use
this for retries over a flaky link. Generate one GUID per logical message and
reuse it on retry; never mint a new GUID for a resend.

> ⚠️ **The other edge of that: a duplicate GUID fails silently.** Idempotent
> means the server replies **success** and does not deliver. There is no error
> code, no reason code, and nothing in the reply distinguishes "delivered" from
> "seen this one already, ignoring it". A client that accidentally reuses GUIDs
> does not get told - its messages simply stop arriving.
>
> The classic way to cause this is an unseeded PRNG. C's `rand()` produces the
> **same sequence on every run** unless you call `srand()`, so a client built
> this way works perfectly the first time it is ever run and silently loses
> every message from the second launch onward.
>
> It is unusually hard to diagnose, because every signal points elsewhere:
>
> - **Typing indicators (813) carry no GUID**, so they keep working - the
>   connection looks healthy and the other end sees you typing.
> - **The server logs the request arriving**, so the framing is plainly fine.
> - **The reply is success**, so even a client that correctly surfaces reply
>   errors has nothing to show.
>
> Seed from something that varies per launch. Wall-clock time alone is not
> enough on a machine whose clock is unset or coarse - which describes a lot of
> vintage hardware and most emulators. Mix in something monotonic: `TickCount()`
> on classic Mac OS, the process id on a host, a counter persisted in your
> preferences file. And prefer a generator whose low bits are trustworthy;
> `rand()` on small libc implementations is often a weak LCG, and every bit of
> a GUID matters.
>
> If your messages vanish with no error, check your GUIDs before anything else.

### 12.2 IM Deliver (811)

Server-initiated notification delivering a message to you. Fields:
`DATA_MESSAGE_GUID`, `DATA_FRIEND_LOGIN` (the **sender**), `DATA_MESSAGE_BODY`,
`DATA_MESSAGE_TIMESTAMP` (`u64` Unix seconds, server-authoritative). Display it,
then send receipts (next).

### 12.3 IM Acknowledge (812)

Request/reply you send to confirm receipt; the server forwards it to the sender.
Fields: `DATA_MESSAGE_GUID`, `DATA_ACK_TYPE` (`1` = `Delivered`, `2` = `Read`),
`DATA_FRIEND_LOGIN` (the **original sender**).

- Send `Delivered` **automatically** as soon as you receive an IM Deliver (811).
- Send `Read` when the user actually views the message.

You also **receive** 812 notifications for messages *you* sent, telling you the
peer delivered/read them - update your conversation UI (single tick → double
tick, etc.). Read receipts survive the sender being offline (forwarded on next
login); a `Delivered` receipt may be dropped if the sender was offline.

### 12.4 IM Typing (813)

Notification, **no reply**, never stored. Fields: `DATA_FRIEND_LOGIN` (recipient
when you send / sender when you receive), `DATA_TYPING_STATE` (`0` = stopped,
`1` = started). Forwarded only to online sessions; silently dropped if the peer
is offline. Send `Started` when the user begins composing, `Stopped` when they
clear the input or send. Debounce - don't emit on every keystroke.

---

## 13. Offline delivery

Store-and-forward applies to **text messages only** (calls, typing, and file
transfers require both peers online and are never queued).

- If you IM an offline friend, IM Send (810) replies `OfflineQueued` (7). Show
  the message as sent-pending in the UI.
- When **you** come online, the server drains *your* undelivered backlog **in
  response to your Get Roster (800)** - it pushes each queued message as an IM
  Deliver (811), oldest first, after the roster sync. So: always send Get Roster
  on login; a client that skips it never receives its backlog. Ack each drained
  message normally (812 `Delivered`), which marks it delivered server-side.
- Pending **read** receipts for messages *you* sent are likewise forwarded to
  you after Get Roster - reconcile them against your sent history.
- The queue is bounded by `MaxOfflinePerRecipient` (default 500). A send beyond
  it returns `QueueFull` (9). Delivered and over-age messages are pruned per
  `OfflineRetentionDays` (default 30).

---

## 14. User-to-user file transfer

Both peers MUST be online and `Accepted`. There is **no** offline file transfer.
Two paths share the same signaling; the data plane differs.

### 14.1 Signaling

1. **File Offer (814)** - request/reply when you offer; notification when you
   receive a peer's offer. Sender fields: `DATA_FRIEND_LOGIN` (recipient),
   `FieldFileName` (201), `FieldFileSize` (207) **or** `DATA_FILESIZE64`
   (0x01F1) for large files, optional `FieldFileTypeString`/
   `FieldFileCreatorString` (205/206), and `DATA_FILE_TRANSFER_GUID` (16 bytes,
   you generate it). The recipient receives the same fields with
   `DATA_FRIEND_LOGIN` set to the sender.
2. **File Accept (815)** - the recipient accepts: `DATA_FILE_TRANSFER_GUID`.
   Forwarded to the sender.
3. **File Decline (816)** - either peer declines/cancels:
   `DATA_FILE_TRANSFER_GUID`, optional `DATA_REASON_CODE`. Forwarded to the
   other peer.
4. **File Ready (817)** - server→client, instructs both peers to begin. For the
   relay path it carries `DATA_FILE_RELAY_REF` (4 bytes). **Each peer gets a
   *distinct* relay ref** mapped to the same session, so the server can tell the
   uploader from the downloader - use the exact ref you were handed.

### 14.2 Relay path (Both - the only path legacy uses)

After you receive File Ready (817) with your `DATA_FILE_RELAY_REF`, open a TCP
connection to the **file-transfer port** (`base + 1`) and present the reference
using the standard HTXF handshake - a 16-byte base header, optionally followed
by fixed-size extension blocks that the flags declare:

| Offset | Size | Field |
|---:|---:|---|
| 0 | 4 | `"HTXF"` (`48 54 58 46`) |
| 4 | 4 | `u32` reference number - the `DATA_FILE_RELAY_REF` you were handed |
| 8 | 4 | `u32` transfer length, 32-bit (the uploader's DATA length; `0` for the downloader) |
| 12 | 4 | `u32` **flags** - `0` for an ordinary transfer; see the flag table further down |
| 16 | 8 | *only when* `HTXF_FLAG_SIZE64` - `u64` transfer length |

```
# a plain 16-byte handshake, no flags set
48 54 58 46  <relay ref as u32>  <size or 0>  00 00 00 00
```

**Offset 12 is flags, not padding.** Older descriptions of the handshake call
it a reserved word, and it is zero in the common case - but the
[large-file extension](../Protocol/Capabilities-Large-File.md#handshake-flags-and-length)
defines it, and a block is present **if and only if** its flag is set. There is
no length field and no other way to detect one, so a reader that ignores the
flags and always consumes 16 bytes will treat an 8-byte length as the first
eight bytes of the file. Send zero when you have no flags to declare; always
read the field.

- The **uploader** then streams the file as a **flattened file object** (`FILP`
  header + `INFO` fork + `DATA` fork; see base protocol, *Flattened File
  Object*); the **downloader** reads the same. The server splices the two TCP
  connections into a live pipe - no disk spooling, because both peers are
  online.
- The reference is **single-use** and expires on the transfer-port handshake
  timeout. Large-file mode (64-bit sizes) applies only if both peers negotiated
  `CAPABILITY_LARGE_FILES`.
- A legacy client implements exactly this and never does anything else for
  transfers.

**Handshake flags on a relay are stricter than on a server transfer.** The
server splices payload; it does not translate framing. Whatever your flags
declare, it must consume before it starts splicing, and the far peer has to be
able to parse what you write.

| Flag | Mask | On a relay |
|---|---|---|
| `HTXF_FLAG_LARGE_FILE` | `0x00000001` | Only if **both** peers negotiated `CAPABILITY_LARGE_FILES`; otherwise the server refuses the transfer. It cannot strip the flag (that would not change the bytes you write) and cannot ignore it (the relay never parses the payload, so it would never find out). |
| `HTXF_FLAG_SIZE64` | `0x00000002` | Same condition - the large-file extension makes it valid only alongside `HTXF_FLAG_LARGE_FILE`. Appends the 8-byte length at offset 16, and the 32-bit length at offset 8 is then set to `0` rather than clamped. |
| `HTXF_FLAG_RESUME` | `0x00000004` | **Always refused**, however capable both peers are. A relay is a live pipe between two online peers: nothing is spooled, so there is no partial and no offset to continue from. Resuming an interrupted user-to-user transfer means a **fresh offer**, not a resumed handshake. (Its 40-byte digest block therefore never appears on a relay handshake, which is why the table above stops at offset 16.) |

The server decides whether large-file mode is permitted once, at File Accept
(815), and applies that to both handshakes - so the answer cannot change between
your connection and your peer's.

**If you are offered a file you cannot frame, decline it.** A recipient whose
session did not get `CAPABILITY_LARGE_FILES` confirmed, offered a file whose size
needs it, SHOULD send File Decline (816) rather than accept and fail at the
handshake. You cannot set the flag, and without the flag you cannot parse what
the sender will send. Declining tells the sender something they can act on while
the transfer still costs nothing. The mirror rule binds senders: never offer a
file you are not authorised to frame.

> The transfer port may be encrypted depending on your session transport - see
> [§14.4](#144-transfer-port-encryption). The HTXF handshake itself is always
> plaintext; encryption (if any) begins right after it.

### 14.3 Direct path (Modern, opt-in)

Used **only** when *both* peers advertise `CAPABILITY_DIRECT_TRANSFER` (check the
peer's `DATA_FRIEND_CAPABILITIES`) **and** the server permits it. The server is
a pure rendezvous broker:

1. After File Accept (815), each peer sends its ICE-style candidate(s) as a File
   Ready (817) carrying `DATA_DIRECT_CANDIDATE` (0x060C, a JSON ICE candidate
   string). This is the **client→server** use of 817 - a no-reply notification;
   the server forwards it to the other peer as a server→client 817 (with
   `DATA_FRIEND_LOGIN` of the originating peer).
2. Both peers attempt a direct (hole-punched) connection.
3. **On any failure** (symmetric NAT, timeout) both peers MUST fall back to the
   [relay path](#142-relay-path-both--the-only-path-legacy-uses). The relay ref
   issued at accept time remains valid as the fallback, so a failed hole-punch is
   transparent.

A modern client MAY carry the direct transfer over a WebRTC data channel
(inheriting DTLS + ICE hole-punching for free); the server signaling is
transport-agnostic.

### 14.4 Transfer-port encryption

How the **relay** file-transfer connection (`base + 1`, or `tlsPort + 1` for TLS)
is protected depends on how the control session is protected. The
**HTXF handshake is always sent in plaintext** on the raw connection; whatever
encryption applies begins immediately *after* the handshake.

| Control session transport | Transfer-port protection |
|---|---|
| Plaintext | Plaintext transfer |
| HOPE stream cipher (RC4/Blowfish) | **Plaintext transfer** (stream mode does not encrypt transfers) |
| HOPE AEAD (ChaCha20-Poly1305) | **Encrypted** with a per-transfer AEAD key (below) |
| TLS | Open the transfer to `tlsPort + 1` inside a **TLS** tunnel; no per-transfer key |

**[Modern] AEAD transfer encryption.** When the control connection uses
ChaCha20-Poly1305, each transfer gets its own key derived from both transport
keys and the 4-byte HTXF reference:

```
ft_base_key  = HKDF-SHA256(ikm = encode_key_256 || decode_key_256,
                           salt = sessionKey, info = "hope-file-transfer")
transfer_key = HKDF-SHA256(ikm = ft_base_key,
                           salt = refNumberBytes,   // the 4-byte HTXF ref, big-endian
                           info = "hope-ft-ref")
```

(`||` is concatenation; both outputs are 32 bytes.)

**The key is per-connection, not per-transfer-session.** One `transfer_key`
protects both directions of *your* connection to the server for the life of the
transfer - but you and your peer were handed **different** relay references, and
the reference is the salt, so the two of you derive two different keys. There is
no shared secret between the peers on this path and there is not meant to be:
the server terminates your protection, and re-frames the plaintext under
whatever your peer's own session negotiated (which may be nothing at all, if
they are a vintage client on a plaintext socket). Do not try to derive your
peer's key, and do not treat a relay transfer as end-to-end encrypted - see the
spec's [The splice is not always a byte
copy](../Protocol/Capabilities-Messaging.md#the-splice-is-not-always-a-byte-copy).

Sequence for an AEAD-protected relay transfer:

1. Open TCP to the transfer port (`base + 1`).
2. Send the HTXF handshake (`"HTXF"` + ref + size + flags, plus any block those
   flags declare - [§14.2](#142-relay-path-both--the-only-path-legacy-uses))
   **in plaintext**.
3. Both ends derive `transfer_key` from the ref number.
4. Initialise ChaCha20-Poly1305 with `transfer_key`.
5. **All subsequent bytes** - the flattened file object (FILP/INFO/DATA forks),
   and for folder transfers the folder-action control bytes - are carried as
   AEAD frames using the **same frame structure, nonce construction, and 16 MiB
   cap as the control connection** ([§7.6](#76-chacha20-poly1305-aead-wire-format-modern)):
   direction byte `0x00` = server→client, `0x01` = client→server, each direction
   with its own counter starting at 0.

So an AEAD messenger reuses its control-connection AEAD frame/nonce code verbatim
for transfers - only the key changes (per-transfer instead of per-session).

> The **direct** path ([§14.3](#143-direct-path-modern-opt-in)) does its own
> thing: over a WebRTC data channel you inherit DTLS; over a raw hole-punched
> socket you define your own protection (or reuse the relay as fallback). The
> per-transfer AEAD scheme above is specifically the relay/HTXF path.

---

## 15. Call signaling and voice (Modern)

**[Modern only]** Voice calls require **both** `CAPABILITY_MESSAGING` (bit 6) and
`CAPABILITY_VOICE` (bit 2) on every participant; friendship is required to ring
someone. A legacy client without bit 2 neither sends nor receives 818–821 and
shows no call UI.

Calls split into two layers:

- **Ring signaling (818–821)** - defined by the messaging extension; tells a
  friend their phone is ringing.
- **Media (600–606)** - the existing
  [Voice Chat extension](../Protocol/Capabilities-Voice.md) (WebRTC
  SFU, G.711 μ-law). The messaging extension adds *only* the ring; once a call
  is accepted the media flow is the voice extension unchanged.

### 15.1 Ring flow

1. **Caller** sends **Call Invite (818)** with one or more `DATA_FRIEND_LOGIN`
   (one for 1:1, several for a group/conference) and **no `DATA_CALL_ID`**.
   Request/reply. **The server allocates the room** and returns `DATA_CALL_ID`
   (4 bytes) in the reply - that is where you learn the call's id, and you need
   it to join media and to hang up.
2. Each invitee's sessions receive Call Invite (818) as a notification:
   `DATA_CALL_ID`, `DATA_FRIEND_LOGIN` (the caller), optional
   `DATA_CALL_MEMBERS` (the packed full invite set - see below). Ring the user.
3. An invitee answers:
   - **Call Accept (819)** - `DATA_CALL_ID`. The server notifies the caller
     (`DATA_CALL_ID` + `DATA_FRIEND_LOGIN` = *which* invitee accepted), and
     sends **Call Cancel (821)** to the accepter's *other* ringing sessions
     (first-accept-wins). The accepter then joins media via **Join Voice Room
     (600)** with `FieldChatID` (114) = the call id.
   - **Call Decline (820)** - `DATA_CALL_ID`, optional `DATA_REASON_CODE`.
     Forwarded to the caller with `DATA_FRIEND_LOGIN` naming the decliner; a
     group call continues for others.
4. **Call Cancel (821)** - sent by the **caller** to withdraw a ring before it is
   answered or to hang up; the server forwards it to unanswered invitees with
   `DATA_FRIEND_LOGIN` = the caller. Only the caller may cancel; a cancel from an
   invitee is ignored (an invitee that wants out sends Decline, or leaves the
   voice room once joined).

> **A cancel with no `DATA_FRIEND_LOGIN` means "you answered this somewhere
> else".** The field names the *other party*, so its absence is not a sloppy
> server - it is the signal that this 821 came from the server silencing your
> own other sessions after one of them accepted (step 3), where there is no
> other party to name. Branch on it: with the field, tell the user the caller
> hung up; without it, just stop ringing. A client that shows "call cancelled"
> for both makes answering on your phone look like a failure on your desktop.

> **Do not invent the call id.** Letting the caller pick it is the natural
> design, and it is wrong here: the call id doubles as the voice room id, so a
> client-chosen id could name a room that already existed - a private chat's
> room, or `0`, the public chat - walking the invitees into it. Sending
> `DATA_CALL_ID` on an invite means something else entirely: *"add these people
> to the call I am already in."* The server honours it only when you are already
> a member of that room and refuses otherwise, so a client that allocates its own
> id cannot place a call at all.

**Inviting more people mid-call.** Send another 818 with the *existing*
`DATA_CALL_ID` plus the new `DATA_FRIEND_LOGIN`s. The new invitees are added to
the room's guest list; the people already in the call stay in it.

**The room is private to its participants.** The server enforces the guest list
on Join Voice Room (600) - being able to name a call's id is not permission to
enter it, for your client or anyone else's. You do not implement anything for
this; it is the reason your id comes from the server.

`DATA_CALL_MEMBERS` (0x060E) packed layout:

```
u16:count
repeat count:
    u16:loginLength
    bytes:login            (negotiated encoding)
    u16:userID             (member's current 16-bit user ID, 0 if not yet joined)
```

### 15.2 Media (summary - see the voice spec for the full detail)

After Join Voice Room (600), the media exchange is standard WebRTC against the
server's SFU on UDP `base + 4`:

- Server reply to 600 carries an **SDP offer** (`DATA_VOICE_SDP` 0x01F5), the
  room codec (`DATA_VOICE_CODEC` 0x01F7 = `"PCMU"`), and current participants
  (`DATA_VOICE_PARTICIPANTS` 0x01F9).
- Reply with **Voice SDP Answer (603)** (`DATA_VOICE_SDP`), then trickle ICE via
  **Voice ICE Candidate (604)** (`DATA_VOICE_ICE` 0x01F6, a JSON
  `RTCIceCandidateInit`); signal end-of-candidates with an empty candidate
  string.
- The server renegotiates (sends new **Voice SDP Offer (602)**) as participants
  join/leave; **Voice Room Status (605)** announces the participant list.
  Track-to-user mapping is by SDP `a=mid:user-{UID}` labels.
- **Voice Mute (606)** toggles your mute (`DATA_VOICE_MUTED` 0x01F8). Clients
  SHOULD join muted by default and treat push-to-talk as a client-side mute
  toggle.
- Codec is **G.711 μ-law (PCMU)**, the mandatory-to-implement WebRTC codec -
  8000 Hz mono, 20 ms frames. Any conformant WebRTC stack (pion, libwebrtc,
  webrtc-rs, browser `RTCPeerConnection`) handles DTLS/SRTP, ICE, and the codec
  for you.

The TCP session is unaffected by WebRTC failure - if a call fails to establish,
text chat continues; surface a "call failed" state and let the user retry.

---

## 16. Reason codes and error handling

`DATA_REASON_CODE` (0x060F, `u16`) is the single enumerated outcome used across
friend, IM, call, and transfer flows. Map each to clear UI:

| Code | Name | Suggested UI handling |
|---:|---|---|
| 0 | `OK` | success |
| 1 | `AccountNotFound` | "No such user." (Also covers a target who blocked you - do not claim "blocked".) |
| 2 | `NotMessagingEnabled` | "That account can't use messaging." |
| 3 | `Blocked` | "You can't message this user." |
| 4 | `AlreadyFriends` | informational; refresh roster |
| 5 | `RequestPending` | "Request already sent." |
| 6 | `NotFriends` | "You must be friends to do that." |
| 7 | `OfflineQueued` | **success** - "Will deliver when they're online." |
| 8 | `OfflineUndeliverable` | "User is offline." (calls/typing/transfer) |
| 9 | `QueueFull` | "Their inbox is full; try later." |
| 10 | `RateLimited` | back off; "Slow down." |
| 11 | `NotDiscoverable` | search returned nothing for that account |
| 12 | `RosterFull` | "Your buddy list is full." |
| 13 | `MessageTooLong` | "That message is too long." - offer to split or trim |

### 16.1 The shape of every reply

Get this wrong once and you get it wrong everywhere, so it is worth stating as a
rule rather than inferring it per transaction:

- **Decide success or failure from the transaction header's `Error code`.
  Never from `DATA_REASON_CODE`.** A non-zero error code is a hard failure; zero
  is success.
- **A failure** carries `FieldError` (100) text written for the user, and
  *usually* `DATA_REASON_CODE`. Some failures have no code that fits - a
  malformed request, a missing field, a subsystem that is off, or an over-long
  message on a server predating code 13. **Show the `FieldError` text** in that
  case; do not fall back to a generic message of your own, and do not treat the
  missing reason code as success.
- **A success** usually carries `DATA_REASON_CODE` = `OK`, but you must not
  require it: Get Roster (800) sends none at all, and IM Send (810) and Get User
  Info (825) succeed while carrying a *non-`OK`* code (`OfflineQueued`,
  `NotFriends`). On a successful reply the reason code describes the outcome, it
  does not contradict it.
- Where a reply carries both a reason code and repeated entries, the reason code
  comes **first** - begin entry parsing at the first `DATA_FRIEND_LOGIN`.

A single `replyError()` helper that checks the header, then pulls reason code
and error text, is all this needs; call it on every reply before you look at
anything else.

---

## 17. Recommended client model

A clean internal model that maps directly onto the protocol:

```
Session
  selfLogin            : string         # your bare identity (you know your own login)
  selfUserID           : u16            # from the login reply (field 103)
  caps                 : bitmask        # confirmed session capabilities
  textMode             : MacRoman | UTF8
  nextTaskID           : u32 counter    # monotonic, skips 0
  pending              : map<taskID, Future>
  limits               : Limits         # from the login reply; defaults if absent

Limits                                  # 0x0620-0x0622, see §7.4
  maxMessageBytes  : u32 = 4096         # ENCODED bytes, not characters
  maxRosterSize    : u32 = 500          # counts every state, blocked included
  maxOfflineQueue  : u32 = 500

Roster
  entries : map<login, RosterEntry>

RosterEntry
  login        : string
  nickname     : string?                # your private alias (824)
  displayName  : string?                # FieldUserName (102); survives them going offline
  state        : Removed|PendingOut|PendingIn|Accepted|Blocked
  presence     : Offline|Online|Away|Invisible|Busy   # last 809 value
  statusText   : string?                # account state: from the 800 snapshot AND 809
  capabilities : bitmask?               # friend's caps; gates call/direct-transfer UI

# Render a row as:  nickname ?? displayName ?? login

Conversation (per friend)
  messages : ordered list of Message
Message
  guid       : bytes[16]
  direction  : in|out
  body       : string
  timestamp  : u64                      # server-stamped on delivery
  state      : sending|queuedOffline|delivered|read   # driven by 810 reply + 812
```

Dispatch table for inbound notifications (task ID 0):

| Type | Handler |
|---|---|
| 801 Roster Entry | upsert/remove `Roster.entries[login]` - **including for changes you made yourself** |
| 804 Friend Request | raise accept/reject prompt |
| 809 Presence Changed | update `entry.presence` / `statusText` / `displayName` |
| 811 IM Deliver | append inbound message; auto-send 812 `Delivered` |
| 812 IM Acknowledge | update your outbound message `state` |
| 813 IM Typing | toggle the typing indicator |
| 814–816 File * | drive the transfer offer UI |
| 817 File Ready | begin relay/direct transfer |
| 818–821 Call * | drive the ring UI |
| 602/604/605 Voice * | feed your WebRTC stack |

---

## 18. Resilience: reconnect, idempotency, dedup

- **Reconnect = full re-login + Get Roster.** Your user ID changes on every
  reconnect, but Logins don't, so the roster and conversations survive. Re-run
  the whole post-login sequence ([§7.5](#75-post-login-the-pure-messenger-shortcut)):
  Set Presence, then Get Roster to re-drain any backlog.
- **Presence is yours to restate, every time.** Nothing about it survives on the
  server - it is live session state, so a reconnected session starts at the
  server's default of online. Keep the user's chosen state in memory across the
  reconnect and re-send it, or someone who dropped while Away silently comes
  back Online to all their friends while their own window still reads "Away".
  This is invisible to the person it happens to, which is why it needs a test
  rather than a look.
- **Don't let one failed step cancel the others.** Presence, the roster sync and
  the outbox resend are independent. A server that refuses the roster can still
  be told your presence and still has your queued messages to accept; bailing
  out of the sequence on the first error leaves a connection that looks healthy
  and is quietly doing neither. Tell the user the buddy list is missing rather
  than showing an empty one.
- **Outbound message idempotency.** Persist a message's GUID *before* sending
  IM Send (810). On reconnect or timeout, resend the same GUID; the server
  dedups by `(GUID, recipient)`. Never allocate a new GUID for a resend, or you
  will create duplicates.
- **Inbound dedup.** You may legitimately receive the same IM Deliver (811) GUID
  twice (e.g. server retry across your reconnect). Key your conversation store by
  GUID and ignore duplicates; still ack each one.
- **Unknown is benign.** Ignore unknown transaction types and unknown fields.
  Servers and peers will add capabilities over time; a forward-compatible client
  never crashes on something new.
- **Presence is soft state.** Don't persist presence across sessions - re-learn
  it from the roster sync and 809s. Persist the *graph* (roster, conversations),
  not the live state.
- **Honour rate limits.** Discovery and IM send are rate-limited; on
  `RateLimited` (10), exponentially back off rather than retrying immediately.

---

## 19. Security considerations for client authors

- **Never derive the *sender* of anything from a client-supplied field.** The
  server attributes senders from the authenticated session. Trust the server's
  `DATA_FRIEND_LOGIN` on inbound 811/812/813/814 as the sender; do not let a
  message body or any field override who sent it in your UI.
- **Block state is intentionally opaque.** A friend request or lookup against a
  user who blocked you returns `AccountNotFound`, not `Blocked`. Do not try to
  reverse-engineer block state, and do not present "you are blocked" - present
  "no such user".
- **So is the online state of anyone who is not your friend.** Discovery never
  reports it, and `DATA_FRIEND_CAPABILITIES` is withheld from non-friends
  precisely because its presence or absence would answer the question by
  implication ([§10](#10-user-discovery-and-profiles)). Do not infer presence
  from which fields a discovery result happens to carry, and do not build a UI
  that hints at it.
- **Validate the HOPE session-key address** (§7.3) to detect MITM/NAT
  redirection.
- **Respect the info-port trust rule** (§4.2): a descriptor may strengthen but
  never silently weaken a saved security setting.
- **Transport confidentiality is the operator's choice.** Over plaintext or the
  relay path, the server sees message and file content (same trust model as
  classic Hotline). If your users need on-wire confidentiality, prefer HOPE
  (AEAD) or TLS, and tell the user what they're getting. The optional direct file
  path can be end-to-end (WebRTC DTLS), but this is never guaranteed (vintage
  peers can't do it) - don't promise E2E unless you negotiated the direct path.
- **The Login is a public handle.** Don't treat knowing a friend's Login as
  authorization for anything; the server enforces friendship/blocks on every
  action regardless of what your UI allows.
- **Cap and sanitise.** Enforce `MaxMessageBytes` client-side too (don't let the
  user compose a message the server will reject). Sanitise status text, nicknames
  and message bodies before rendering (they come from other users).

---

## 20. Conformance checklist

A client is a conforming messenger when it:

- [ ] Completes the TRTP handshake and (Modern) honours the info-port descriptor.
- [ ] Advertises `CAPABILITY_MESSAGING` (bit 6) and `CAPABILITY_MESSENGER_SESSION`
      (bit 8) in Login, and **disables all messaging UI** if the reply doesn't
      confirm bit 6.
- [ ] Bitwise-inverts **both** the login and the password in a legacy plaintext
      Login (107), and uses the un-inverted password for every HOPE MAC.
- [ ] Implements the transaction framing: 20-byte header, field list, big-endian
      integers honouring `fieldSize`, fixed-width messaging fields.
- [ ] Dispatches by is-reply flag + task ID; handles task-ID-0 notifications as
      events; treats bidirectional types correctly by direction.
- [ ] Ignores unknown fields and unknown transaction types.
- [ ] Transcodes string fields per the negotiated encoding (Mac Roman or UTF-8).
- [ ] Sends Set Presence (808) and then Get Roster (800) after **every** login,
      reconnects included - restating the user's chosen presence, which the
      server does not remember, before the readiness signal that drains the
      offline backlog.
- [ ] Captures `DATA_MAX_MESSAGE_BYTES` / `DATA_MAX_ROSTER_SIZE` /
      `DATA_MAX_OFFLINE_QUEUE` from the login reply, falls back to the defaults
      when absent, and pre-validates against them in **encoded bytes**.
- [ ] Parses repeated entries by the `DATA_FRIEND_LOGIN` delimiter, attributes
      each `FieldUserName` (102) to its own entry group, and accepts both the
      inline-reply and streamed-801 roster forms.
- [ ] Stores the friend's `FieldUserName` **and** `DATA_PRESENCE_STATUS_TEXT`
      from the roster snapshot as well as from 809 (both are account state and
      outlive the session), and shows alias → published name → Login, in that
      order, without appending the Login.
- [ ] Implements add/remove/block/unblock/nickname and the friend-request
      accept/reject flow, treating removal as **mutual** and applying the 801
      echoes to its own sessions idempotently.
- [ ] **Replaces** a roster entry with the 801 group rather than merging into
      it, so clearing an alias or a status actually clears it, and uses one
      routine for snapshot entries and deltas alike.
- [ ] Sends `DATA_PRESENCE_STATUS_TEXT` present-but-empty to clear a status,
      never omitted.
- [ ] Decides success from the header error code, shows `FieldError` (100) text
      when a failure carries no reason code, and never requires
      `DATA_REASON_CODE` = `OK` on a success.
- [ ] Implements IM send/deliver/ack/typing, generates a UUIDv4 GUID per message,
      retries idempotently with the same GUID, and auto-sends `Delivered` plus
      `Read` receipts.
- [ ] Handles `OfflineQueued` as success and reconciles the drained backlog and
      pending read receipts after Get Roster.
- [ ] Gates per-friend call/direct-transfer UI on `DATA_FRIEND_CAPABILITIES`.
- [ ] Implements the relay file-transfer path (HTXF + flattened file object),
      with the correct transfer-port protection for its transport (plaintext /
      AEAD per-transfer key / TLS to `tlsPort + 1`).
- [ ] Refuses `HTXF_FLAG_RESUME` on a relay, sets `HTXF_FLAG_LARGE_FILE` /
      `HTXF_FLAG_SIZE64` only when both peers hold `CAPABILITY_LARGE_FILES`, and
      declines an offer it cannot frame instead of failing at the handshake.
- [ ] **(Modern)** Implements at least one real transport - TLS (verified) or
      HOPE AEAD (ChaCha20-Poly1305 frames + nonce discipline) - plus UTF-8,
      direct transfer with relay fallback, and voice/conference calls (818–821
      ring + 600–606 media), taking `DATA_CALL_ID` **from the server's reply**
      rather than allocating one.
- [ ] Maps every `DATA_REASON_CODE` to clear UI and never claims "blocked" for a
      `AccountNotFound`.

---

## Appendix A - Enumerations

**`DATA_PRESENCE_STATE` (0x0602, u16):** `0` Offline (server→client only,
disconnected *or* invisible) · `1` Online · `2` Away · `3` Invisible · `4` Busy.

**`DATA_ROSTER_STATE` (0x0604, u16):** `0` Removed (801 delta sentinel) · `1`
PendingOut · `2` PendingIn · `3` Accepted · `4` Blocked.

**`DATA_ACK_TYPE` (0x0608, u16):** `1` Delivered · `2` Read.

**`DATA_TYPING_STATE` (0x0609, u16):** `0` Stopped · `1` Started.

**`DATA_DISCOVERABLE` (0x0611, u16):** `0` Unlisted (default) · `1`
Directory-searchable.

**`DATA_PROFILE_GENDER` (0x0618, u16):** `0` Unspecified · `1` Female · `2`
Male · `3` Other. An unrecognised value is stored as unspecified, never
rejected.

**`DATA_REASON_CODE` (0x060F, u16):** see [§16](#16-reason-codes-and-error-handling).

---

## Appendix B - Field ID quick table

Messaging fields (0x0600 block):

| Hex | Dec | Name | Type |
|---|---:|---|---|
| 0x0600 | 1536 | `DATA_FRIEND_LOGIN` | string (entry delimiter) |
| 0x0601 | 1537 | `DATA_FRIEND_NICKNAME` | string |
| 0x0602 | 1538 | `DATA_PRESENCE_STATE` | u16 |
| 0x0603 | 1539 | `DATA_PRESENCE_STATUS_TEXT` | string |
| 0x0604 | 1540 | `DATA_ROSTER_STATE` | u16 |
| 0x0605 | 1541 | `DATA_MESSAGE_GUID` | bytes[16] |
| 0x0606 | 1542 | `DATA_MESSAGE_BODY` | string |
| 0x0607 | 1543 | `DATA_MESSAGE_TIMESTAMP` | u64 (Unix sec) |
| 0x0608 | 1544 | `DATA_ACK_TYPE` | u16 |
| 0x0609 | 1545 | `DATA_TYPING_STATE` | u16 |
| 0x060A | 1546 | `DATA_FILE_TRANSFER_GUID` | bytes[16] |
| 0x060B | 1547 | `DATA_FILE_RELAY_REF` | bytes[4] |
| 0x060C | 1548 | `DATA_DIRECT_CANDIDATE` | string (JSON ICE) |
| 0x060D | 1549 | `DATA_CALL_ID` | bytes[4] |
| 0x060E | 1550 | `DATA_CALL_MEMBERS` | binary (packed) |
| 0x060F | 1551 | `DATA_REASON_CODE` | u16 |
| 0x0610 | 1552 | `DATA_REQUEST_NOTE` | string |
| 0x0611 | 1553 | `DATA_DISCOVERABLE` | u16 |
| 0x0612 | 1554 | `DATA_SEARCH_QUERY` | string |
| 0x0613 | 1555 | `DATA_FRIEND_CAPABILITIES` | u16 |
| 0x0614 | 1556 | `DATA_PROFILE_NICKNAME` | string |
| 0x0615 | 1557 | `DATA_PROFILE_FIRST_NAME` | string |
| 0x0616 | 1558 | `DATA_PROFILE_LAST_NAME` | string |
| 0x0617 | 1559 | `DATA_PROFILE_EMAIL` | string |
| 0x0618 | 1560 | `DATA_PROFILE_GENDER` | u16 |
| 0x0619 | 1561 | `DATA_PROFILE_BIRTHDATE` | bytes[4] (`u16` year, `u8` month, `u8` day) |
| 0x061A | 1562 | `DATA_PROFILE_COUNTRY` | string (ISO 3166-1 alpha-2) |
| 0x061B | 1563 | `DATA_PROFILE_POSTCODE` | string |
| 0x061C | 1564 | `DATA_PROFILE_LANGUAGE` | string (ISO 639-1, repeated ×3 max) |
| 0x0620 | 1568 | `DATA_MAX_MESSAGE_BYTES` | u32 (login reply only) |
| 0x0621 | 1569 | `DATA_MAX_ROSTER_SIZE` | u32 (login reply only) |
| 0x0622 | 1570 | `DATA_MAX_OFFLINE_QUEUE` | u32 (login reply only) |

Reused base/extension fields:

| Hex | Dec | Name | Type |
|---|---:|---|---|
| 0x0064 | 100 | `FieldError` | string |
| 0x0065 | 101 | `FieldData` | string/binary |
| 0x0066 | 102 | `FieldUserName` | string |
| 0x0067 | 103 | `FieldUserID` | integer (u16) |
| 0x0068 | 104 | `FieldUserIconID` | integer |
| 0x0069 | 105 | `FieldUserLogin` | string (login) |
| 0x006A | 106 | `FieldUserPassword` | string |
| 0x0072 | 114 | `FieldChatID` | integer (u32) |
| 0x00A0 | 160 | `FieldVersion` | integer |
| 0x00A2 | 162 | `FieldServerName` | string |
| 0x00C9 | 201 | `FieldFileName` | string |
| 0x00CD | 205 | `FieldFileTypeString` | string |
| 0x00CE | 206 | `FieldFileCreatorString` | string |
| 0x00CF | 207 | `FieldFileSize` | integer |
| 0x01F0 | 496 | `DATA_CAPABILITIES` | bitmask |
| 0x01F1 | 497 | `DATA_FILESIZE64` | u64 (large files) |
| 0x01F5 | 501 | `DATA_VOICE_SDP` | string |
| 0x01F6 | 502 | `DATA_VOICE_ICE` | string (JSON) |
| 0x01F7 | 503 | `DATA_VOICE_CODEC` | string |
| 0x01F8 | 504 | `DATA_VOICE_MUTED` | u16 |
| 0x01F9 | 505 | `DATA_VOICE_PARTICIPANTS` | binary (packed) |
| 0x0E01 | 3585 | HOPE App ID | bytes[4] |
| 0x0E02 | 3586 | HOPE App String | string |
| 0x0E03 | 3587 | HOPE Session Key | bytes[64] |
| 0x0E04 | 3588 | HOPE MAC Algorithm | list |
| 0x0EC1–0x0ECA | 3777–3786 | HOPE cipher/mode/IV/checksum/compression | various |

---

## Appendix C - Transaction ID quick table

Messaging (800 block):

| ID | Hex | Name | Direction |
|---:|---|---|---|
| 800 | 0x0320 | Get Roster | C→S (req/reply) |
| 801 | 0x0321 | Roster Entry | S→C (notify) |
| 802 | 0x0322 | Add Friend | C→S (req/reply) |
| 803 | 0x0323 | Remove Friend | C→S (req/reply) |
| 804 | 0x0324 | Friend Request | S→C (notify) |
| 805 | 0x0325 | Friend Response | C→S (req/reply) |
| 806 | 0x0326 | Block User | C→S (req/reply) |
| 807 | 0x0327 | Unblock User | C→S (req/reply) |
| 808 | 0x0328 | Set Presence | C→S (req/reply) |
| 809 | 0x0329 | Presence Changed | S→C (notify) |
| 810 | 0x032A | IM Send | C→S (req/reply) |
| 811 | 0x032B | IM Deliver | S→C (notify) |
| 812 | 0x032C | IM Acknowledge | bidirectional |
| 813 | 0x032D | IM Typing | bidirectional (notify) |
| 814 | 0x032E | File Offer | bidirectional |
| 815 | 0x032F | File Accept | bidirectional |
| 816 | 0x0330 | File Decline | bidirectional |
| 817 | 0x0331 | File Ready | S→C (relay) / bidirectional (direct) |
| 818 | 0x0332 | Call Invite | bidirectional |
| 819 | 0x0333 | Call Accept | bidirectional |
| 820 | 0x0334 | Call Decline | bidirectional |
| 821 | 0x0335 | Call Cancel | bidirectional |
| 822 | 0x0336 | Find User | C→S (req/reply) |
| 823 | 0x0337 | User Search | C→S (req/reply) |
| 824 | 0x0338 | Set Friend Nickname | C→S (req/reply) |
| 825 | 0x0339 | Get User Info | C→S (req/reply) |
| 826 | 0x033A | Set User Info | C→S (req/reply) |

Reused: Login **107**, Show Agreement **109**, Agreed **121**; Join/Leave Voice
Room **600/601**, Voice SDP Offer/Answer **602/603**, Voice ICE **604**, Voice
Room Status **605**, Voice Mute **606**.

---

## Appendix D - Worked byte-level example: IM Send

Sending the message `"hi"` to friend `alice` with a GUID
`0123456789abcdef0123456789abcdef`, using task ID `7`, UTF-8 session.

**Payload** (field list):

```
field count                              00 03

# DATA_FRIEND_LOGIN (0x0600), "alice" (5 bytes UTF-8)
06 00  00 05  61 6C 69 63 65

# DATA_MESSAGE_GUID (0x0605), 16 bytes
06 05  00 10  01 23 45 67 89 AB CD EF 01 23 45 67 89 AB CD EF

# DATA_MESSAGE_BODY (0x0606), "hi" (2 bytes)
06 06  00 02  68 69
```

Payload length = 2 + (4+5) + (4+16) + (4+2) = **37 bytes** (0x25).

**Header** (20 bytes): flags 0, is-reply 0, type 810 (0x032A), task ID 7,
error 0, total size 37, data size 37:

```
00 00  03 2A  00 00 00 07  00 00 00 00  00 00 00 25  00 00 00 25
```

**On the wire**: the 20-byte header immediately followed by the 37-byte payload
(57 bytes total). The server replies with is-reply 1, **task ID 7**, and either
error 0 + `DATA_REASON_CODE = OK`/`OfflineQueued`, or a non-zero error + a
failure `DATA_REASON_CODE`.

Note what the reply's *type* is not: the reference server sends `0`, not
`0x032A`. Task ID 7 is the only thing tying that reply to this request, which is
why [§5.4](#54-task-ids-replies-and-notifications) insists you route on it.

To **retry** after a timeout, resend the identical bytes (same GUID) - the server
dedups by `(GUID, recipient)`, so the retry is idempotent.
