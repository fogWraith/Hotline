# Chat History Extension

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document describes the chat history extension to the Hotline protocol. It adds server-side persistence of chat messages with cursor-based pagination, allowing clients to retrieve historical messages in batches — similar to how Discord loads channel history as a user scrolls. The extension is entirely optional for both server and client, uses the standard capability negotiation mechanism, and is designed to be storage-agnostic so that any Hotline server implementation can adopt it regardless of backend technology.

For the general capability negotiation mechanism, see [DATA_CAPABILITIES](https://github.com/fogWraith/Hotline/blob/main/Docs/Protocol/Capabilities.md).

## Table of Contents

- [Background](#background)
- [Concepts](#concepts)
  - [Channels vs Private Chats](#channels-vs-private-chats)
  - [Cursor-Based Pagination](#cursor-based-pagination)
  - [Message IDs](#message-ids)
  - [Legacy Broadcast](#legacy-broadcast)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
  - [Capability Bit](#capability-bit)
  - [Retention Policy Advertisement](#retention-policy-advertisement)
  - [Server Configuration](#server-configuration)
- [Data Objects](#data-objects)
- [Transaction Types](#transaction-types)
  - [Get Chat History (700)](#get-chat-history-700)
- [History Entry Format](#history-entry-format)
  - [Fixed Fields](#fixed-fields)
  - [Variable Fields](#variable-fields)
  - [Optional Sub-Fields (Mini-TLV)](#optional-sub-fields-mini-tlv)
  - [Flags](#flags)
  - [Text Encoding](#text-encoding)
  - [Parsing Algorithm](#parsing-algorithm)
- [Channel Semantics](#channel-semantics)
  - [Channel 0: Public Chat](#channel-0-public-chat)
  - [Named Channels (Future)](#named-channels-future)
  - [Private Chat Privacy](#private-chat-privacy)
- [Access Privileges](#access-privileges)
- [Legacy Broadcast Behaviour](#legacy-broadcast-behaviour)
- [Storage](#storage)
  - [Requirements](#requirements)
  - [Encryption at Rest](#encryption-at-rest)
- [Reserved Transaction Types](#reserved-transaction-types)
- [Implementation Notes](#implementation-notes)
- [Example: Login with Chat History Negotiation](#example-login-with-chat-history-negotiation)
- [Example: Get Chat History Request and Reply](#example-get-chat-history-request-and-reply)
- [Example: Scrollback Pagination](#example-scrollback-pagination)
- [Example: Catching Up After Reconnect](#example-catching-up-after-reconnect)

---

## Background

The standard Hotline protocol treats chat as a real-time stream — messages are broadcast to connected clients and then forgotten. A user who connects to a busy server sees nothing before their arrival. If they disconnect and reconnect, the conversation is gone.

Some clients (notably Hotline Navigator) compensate by storing chat history locally in an encrypted vault. This works for a single device, but the history is invisible to other clients, other devices, and new installations. There is no shared record of the conversation.

The chat history extension moves persistence to the server. When enabled, the server records public chat messages and serves them to clients on demand. Clients that support the extension can scroll back through history, catch up after being offline, and present the server's chat as a continuous conversation rather than an ephemeral stream.

The extension is designed with three priorities:

1. **Backward compatibility.** Clients that do not support chat history see no change. The live chat broadcast path (`TRAN_CHAT_MSG`, type 106) is completely unchanged.
2. **Adoption simplicity.** The wire protocol follows established Hotline patterns — TLV fields, packed binary entries, capability bitmask negotiation. The spec does not prescribe a storage backend, so implementers can use SQLite, flat files, or anything else.
3. **Future extensibility.** The entry format includes a mini-TLV mechanism for optional sub-fields, and the channel ID namespace reserves space for Discord-style named channels in a future version.

---

## Concepts

### Channels vs Private Chats

Hotline has two kinds of multi-user conversation:

- **Public chat** — a single global room that all connected users share. Messages are broadcast to everyone with read-chat permission.
- **Private chats** — ephemeral rooms created on the fly via invitation. They have a 4-byte chat ID, and they vanish when the last member leaves.

This extension introduces a third concept: **channels**. A channel is a persistent, named conversation space — like a Discord channel. Channel 0 is always the public chat. Channels 1+ are reserved for future named channels (e.g., `#general`, `#dev`, `#music`).

Channels and private chats use **separate ID namespaces** to avoid ambiguity:

| Conversation Type | ID Field | Persistence | History |
|---|---|---|---|
| Public chat | `DATA_CHANNEL_ID` = 0 | Permanent | Server-side (this extension) |
| Named channels (future) | `DATA_CHANNEL_ID` = 1+ | Permanent | Server-side (this extension) |
| Private chats | `DATA_CHATID` (0x0072) | Ephemeral | Client-side only |

The distinction is intentional. Private chats carry an expectation of transience — users pull others aside for a temporary conversation. Server-side logging of those conversations would violate that expectation. See [Private Chat Privacy](#private-chat-privacy).

### Cursor-Based Pagination

Chat history uses **cursor-based pagination** rather than page numbers. A cursor is a message ID that marks a position in the history. The client says "give me 50 messages before this cursor" or "give me 50 messages after this cursor."

This approach has several advantages over offset-based pagination (page 1, page 2, ...):

- **Stable under writes.** New messages arriving don't shift the position of older pages. With offset pagination, a message posted between page requests would cause a message to appear on two pages or be skipped entirely.
- **Efficient for append-heavy workloads.** The server can use the cursor to seek directly to the right position, regardless of storage backend.
- **Stateless.** The server doesn't track where each client is in their scroll position. The cursor in each request is self-contained.

Clients maintain cursors locally — typically by remembering the message ID of the oldest or newest message they've seen, and using it as the cursor in the next request.

### Message IDs

Every persisted message receives a **message ID** — a 64-bit unsigned integer assigned by the server. The only guarantee is that IDs are **monotonically increasing**: a newer message always has a higher ID than an older one.

Message IDs are **opaque**. Clients MUST NOT interpret their values — they could be sequential counters, timestamp-derived snowflakes, or any other scheme. Different server implementations will use different generation strategies. The client's only valid operations on a message ID are:

- Comparing two IDs to determine relative ordering (higher = newer)
- Passing an ID back to the server as a pagination cursor

### Legacy Broadcast

When a client connects without advertising chat history support (bit 4 not set), the server MAY still send recent messages — but using the existing `TRAN_CHAT_MSG` (106) transactions. To the client, these look identical to live chat messages. This is the "legacy broadcast" path.

The legacy broadcast is entirely implementation-defined — the server decides whether to send it, how many messages to include, and when to send them. Some clients (like Hotline Navigator) have heuristics to detect and label these replayed messages. Other clients will simply display them as if 30 people all typed at once.

This is a pragmatic bridge, not a first-class feature. Clients that want a good history experience should implement the full extension.

---

## Compatibility and Negotiation

### Capability Bit

| Bit | Mask | Name | Description |
|---|---|---|---|
| 4 | `0x0010` | `CAPABILITY_CHAT_HISTORY` | Client supports server-side chat history retrieval |

This bit is defined in the `DATA_CAPABILITIES` bitmask (field `0x01F0`). See [DATA_CAPABILITIES](https://github.com/fogWraith/Hotline/blob/main/Docs/Protocol/Capabilities.md) for the general negotiation flow.

**Negotiation:**

1. Client sets bit 4 in `DATA_CAPABILITIES` during Login (107).
2. Server checks its configuration. If chat history is enabled, it echoes bit 4 in the login reply.
3. If the server does not echo bit 4, chat history is unavailable. The client MUST NOT send `TRAN_GET_CHAT_HISTORY` (700) transactions.

Clients that do not set bit 4 are unaffected — they participate in live chat normally and never receive history-related fields.

When multiple extensions are active (e.g., large files + text encoding + chat history), the capability bitmask combines the bits: `0x0013` = bits 0, 1, and 4.

### Retention Policy Advertisement

The server MAY include retention policy hints in the login reply alongside the echoed `DATA_CAPABILITIES`:

| Field | ID (hex) | Type | Description |
|---|---|---|---|
| `DATA_HISTORY_MAX_MSGS` | `0x0F07` | uint32 | Maximum number of messages the server retains. `0` = unlimited. |
| `DATA_HISTORY_MAX_DAYS` | `0x0F08` | uint32 | Maximum number of days the server retains messages. `0` = unlimited. |

These fields are **informational only**. They tell the client what the server's retention policy is — not how many messages currently exist. A server configured for "30 days" might have only 2 days of history if it was recently set up.

Clients MAY use these values for UI hints (e.g., "This server keeps 30 days of chat history") but MUST NOT depend on them for correctness. The authoritative signal for "no more messages" is `DATA_HISTORY_HAS_MORE = 0` in a Get Chat History reply.

If both fields are absent, the client should not assume any particular retention policy.

### Server Configuration

The server operator controls chat history via configuration. Recommended settings (implementation-defined):

| Setting | Type | Default | Description |
|---|---|---|---|
| `Enabled` | bool | `false` | Master switch for chat history persistence |
| `MaxMessages` | int | `0` | Maximum messages retained. `0` = unlimited |
| `MaxDays` | int | `0` | Maximum retention period in days. `0` = unlimited |
| `LegacyBroadcast` | bool | `true` | Send recent messages to non-history clients on connect |
| `LegacyBroadcastCount` | int | `30` | Number of messages to broadcast to legacy clients |
| `EncryptionKey` | string | `""` | Key or key file path for encryption at rest |

---

## Data Objects

All new data objects use field IDs in the `0x0F01`–`0x0F1F` range, following the convention established by the HOPE extension (`0x0E01`–`0x0ECA`) of placing protocol extensions in the high field ID space. All multi-byte integers are unsigned big-endian (network byte order) unless otherwise noted.

| ID (hex) | Decimal | Name | Type | Description |
|---|---|---|---|---|
| `0x0F01` | 3841 | `DATA_CHANNEL_ID` | uint32 | Persistent channel identifier. `0` = public chat, `1`+ = named channels (reserved). |
| `0x0F02` | 3842 | `DATA_HISTORY_BEFORE` | uint64 | Cursor: return messages with IDs less than this value. |
| `0x0F03` | 3843 | `DATA_HISTORY_AFTER` | uint64 | Cursor: return messages with IDs greater than this value. |
| `0x0F04` | 3844 | `DATA_HISTORY_LIMIT` | uint16 | Maximum number of messages to return in a single response. |
| `0x0F05` | 3845 | `DATA_HISTORY_ENTRY` | binary | A single chat history entry in packed binary format. See [History Entry Format](#history-entry-format). |
| `0x0F06` | 3846 | `DATA_HISTORY_HAS_MORE` | uint8 | `1` if more messages exist beyond this batch, `0` otherwise. |
| `0x0F07` | 3847 | `DATA_HISTORY_MAX_MSGS` | uint32 | Server retention policy: maximum messages. `0` = unlimited. Login reply only. |
| `0x0F08` | 3848 | `DATA_HISTORY_MAX_DAYS` | uint32 | Server retention policy: maximum days. `0` = unlimited. Login reply only. |
| `0x0F09`–`0x0F1F` | 3849–3871 | *Reserved* | — | Reserved for future channel management fields (e.g., channel name, topic, channel list entries). |

---

## Transaction Types

| ID | Name | Direction | Description |
|---|---|---|---|
| 700 | Get Chat History | Client → Server | Request a batch of historical messages |
| 701 | *Get Chat History Info* | — | Reserved: query channel listing, metadata, or retention details |
| 702 | *Channel Management* | — | Reserved: create, delete, or modify named channels |
| 703 | *Delete Chat History Message* | — | Reserved: administrator removal of a message (tombstone) |
| 704 | *Edit Chat History Message* | — | Reserved: administrator or user editing of a message |
| 705–709 | *Unassigned* | — | Reserved: future chat history extensions |

Transaction IDs 700–709 are chosen to avoid collision with existing Hotline transaction types (base protocol: 101–355, keepalive: 500, voice: 600–606, GIF icons: 1861–1864).

### Get Chat History (700)

**Client → Server (request/reply)**

The client requests a batch of historical chat messages for a specific channel. The server replies with zero or more history entries and a "has more" indicator.

**Request fields:**

| Field | ID | Type | Required | Description |
|---|---|---|---|---|
| Channel ID | `0x0F01` | uint32 | Yes | Channel to query. `0` = public chat. |
| History Before | `0x0F02` | uint64 | No | Return messages with IDs strictly less than this value. |
| History After | `0x0F03` | uint64 | No | Return messages with IDs strictly greater than this value. |
| History Limit | `0x0F04` | uint16 | No | Maximum messages to return. Server default: 50. |

**Cursor rules:**

The two cursor fields control which portion of history is returned. Think of the message timeline as a number line where message IDs increase to the right:

```
oldest                                              newest
  ◄─────────────────────────────────────────────────────►
  100   200   300   400   500   600   700   800   900
```

| Cursor(s) Present | Behaviour | Typical Use Case |
|---|---|---|
| Neither | Return the most recent messages (up to limit) | Initial load — "show me what's been happening" |
| `BEFORE` only | Messages older than the cursor | Scrolling back — "show me older messages" |
| `AFTER` only | Messages newer than the cursor | Catching up — "what did I miss since I was last here?" |
| Both | Messages in the range (AFTER, BEFORE) | Range query — "fill in this gap in my local cache" |

```
Example: BEFORE=600, AFTER=200, LIMIT=50

  100   200   300   400   500   600   700   800   900
         │◄────── this range ──────►│
         (exclusive)                (exclusive)

  Returns messages 201–599 (up to 50), oldest-first.
```

**Reply fields:**

| Field | ID | Type | Required | Description |
|---|---|---|---|---|
| History Entry | `0x0F05` | binary | Repeated (0–N) | One field per message, packed binary. See [History Entry Format](#history-entry-format). |
| Has More | `0x0F06` | uint8 | Yes | `1` if more messages exist beyond this batch, `0` otherwise. |

Reply entries are always ordered **oldest-first** (ascending message ID) regardless of which cursor was used. This simplifies client rendering — messages can be appended directly to the chat view in the order received.

If no messages match the query, the reply contains zero `DATA_HISTORY_ENTRY` fields and `DATA_HISTORY_HAS_MORE` = 0.

**`has_more` semantics:** The `DATA_HISTORY_HAS_MORE` flag indicates whether additional messages exist in the **direction of the query**:

| Query Type | `has_more = 1` means |
|---|---|
| No cursors (latest) | Older messages exist beyond this batch |
| `BEFORE` only | Older messages exist before the oldest returned entry |
| `AFTER` only | Newer messages exist after the newest returned entry |
| Both cursors (range) | More messages exist within the specified range |

**Limit clamping:** Servers MAY enforce a maximum limit (e.g., 200) regardless of the client's requested value. When the server reduces the limit, it simply returns fewer entries than requested — no error is raised. Clients MUST NOT assume the number of returned entries equals their requested limit; always check `DATA_HISTORY_HAS_MORE`.

**Error conditions:**

| Condition | Behaviour |
|---|---|
| Client lacks `ACCESS_READ_CHAT_HISTORY` permission | Error reply with descriptive text |
| Channel ID not supported by server | Error reply with descriptive text |
| Client did not negotiate capability bit 4 | Server MAY reject or ignore the transaction |

---

## History Entry Format

Each `DATA_HISTORY_ENTRY` (field `0x0F05`) contains a single chat message in packed binary format. The format follows the same conventions as other packed structures in the Hotline protocol (e.g., `DATA_FILE_NAME_WITH_INFO`, user info payloads) — fixed-size header fields followed by length-prefixed variable data.

### Fixed Fields

| Offset | Size | Field | Type | Description |
|---|---|---|---|---|
| 0 | 8 | Message ID | uint64 | Server-assigned opaque identifier. Monotonically increasing. |
| 8 | 8 | Timestamp | int64 (signed) | Unix epoch seconds (UTC) when the message was received by the server. |
| 16 | 2 | Flags | uint16 | Bitfield. See [Flags](#flags). |
| 18 | 2 | Icon ID | uint16 | Sender's icon ID at the time the message was sent. `0` = no icon. |
| 20 | 2 | Nick Length | uint16 | Length of the sender's nickname in bytes. |

### Variable Fields

| Offset | Size | Field | Description |
|---|---|---|---|
| 22 | Nick Length | Nick | Sender's display nickname. |
| 22 + Nick Length | 2 | Message Length | uint16. Length of the message body in bytes. |
| 24 + Nick Length | Message Length | Message | The chat message text. |

**Total minimum size:** 24 bytes (empty nick and empty message).

### Optional Sub-Fields (Mini-TLV)

After the message body, zero or more **optional sub-fields** MAY be present. This mechanism allows future extensions to add data to history entries without breaking existing parsers.

Each sub-field is encoded as:

| Size | Field | Description |
|---|---|---|
| 2 | Sub-Type | uint16. Identifies the sub-field. |
| 2 | Sub-Length | uint16. Length of Sub-Data in bytes. |
| Sub-Length | Sub-Data | The sub-field's payload. |

Clients determine whether sub-fields exist by comparing bytes consumed against the field's total data length (known from the outer TLV field header). If bytes remain after parsing the message body, those bytes are sub-fields.

**Parsing rule:** Clients MUST skip sub-fields with unrecognised Sub-Type values, using Sub-Length to advance to the next sub-field. This ensures forward compatibility — a v1 client safely ignores v2 sub-fields.

**No sub-field types are defined in this version.** The mechanism is reserved for future use. Possible future sub-fields include:

| Sub-Type (proposed) | Name | Size | Description |
|---|---|---|---|
| `0x0001` | Nick Color | 4 | `0x00RRGGBB` — sender's nickname color |
| `0x0002` | Reply-To ID | 8 | Message ID of the message being replied to |
| `0x0003` | Account Login | variable | The sender's account login name (vs display nick) |

These are illustrative examples, not commitments. Actual sub-field types will be defined in future revisions of this specification.

### Flags

| Bit | Mask | Name | Description |
|---|---|---|---|
| 0 | `0x0001` | `is_action` | Message was a `/me` emote (displayed as `*** nick does something`). Corresponds to `DATA_CHATOPTIONS` (`0x006E`) value `1` on the live chat path — the server sets this flag when recording an emote. |
| 1 | `0x0002` | `is_server_msg` | Message originated from the server (e.g., admin broadcast), not a user. |
| 2 | `0x0004` | `is_deleted` | Message has been removed by an administrator (tombstone). See below. |
| 3–15 | — | *Reserved* | MUST be zero. Clients MUST ignore unknown flag bits. |

**Tombstoned entries** (bit 2 set): The message has been deleted by an administrator. The `message_id` and `timestamp` are preserved — this is critical for cursor stability, as removing entries entirely would break pagination for clients mid-scroll. The `nick` and `message` fields MAY have zero length. Clients SHOULD display a placeholder such as "[message removed]" and SHOULD NOT attempt to recover or display the original content.

### Text Encoding

The `nick` and `message` fields are encoded in the text encoding negotiated for the requesting connection — either Mac Roman or UTF-8, matching the `CAPABILITY_TEXT_ENCODING` (bit 1) negotiation.

Servers store messages in their internal encoding and transcode on retrieval, exactly as they do for live `TRAN_CHAT_MSG` (106) broadcasts. A UTF-8 client receives UTF-8 history entries; a Mac Roman client receives Mac Roman entries, regardless of what encoding the original sender used.

**Lossy transcoding:** When converting UTF-8 text to Mac Roman, characters that have no Mac Roman representation MUST be replaced with `?` (0x3F). Servers MUST NOT silently drop unmappable characters, insert multi-byte substitution sequences, or return an error. This matches the established behaviour of the existing Hotline text encoding path and ensures consistent output across server implementations.

### Parsing Algorithm

The following pseudocode demonstrates how to parse a `DATA_HISTORY_ENTRY`:

```
function parse_history_entry(data, data_len):
    // Fixed header requires at least 22 bytes
    if data_len < 22:
        return error("entry too short for fixed header")

    message_id  = read_uint64(data, 0)
    timestamp   = read_int64(data, 8)
    flags       = read_uint16(data, 16)
    icon_id     = read_uint16(data, 18)
    nick_len    = read_uint16(data, 20)

    // Validate nick fits: 22 + nick_len + 2 (msg_len field) <= data_len
    if 22 + nick_len + 2 > data_len:
        return error("entry too short for nick + msg_len")

    nick        = read_bytes(data, 22, nick_len)
    msg_len     = read_uint16(data, 22 + nick_len)

    // Validate message fits
    if 24 + nick_len + msg_len > data_len:
        return error("entry too short for message body")

    message     = read_bytes(data, 24 + nick_len, msg_len)

    // Optional sub-fields (remaining bytes after message)
    offset = 24 + nick_len + msg_len
    sub_fields = []
    while offset + 4 <= data_len:
        sub_type = read_uint16(data, offset)
        sub_len  = read_uint16(data, offset + 2)
        if offset + 4 + sub_len > data_len:
            break  // malformed sub-field, stop parsing
        sub_data = read_bytes(data, offset + 4, sub_len)
        sub_fields.append({type: sub_type, data: sub_data})
        offset += 4 + sub_len

    return {message_id, timestamp, flags, icon_id,
            nick, message, sub_fields}
```

---

## Channel Semantics

### Channel 0: Public Chat

Channel 0 is the public chat room — the same global conversation that `TRAN_CHAT_MSG` (106) broadcasts to. Every server that enables chat history MUST support channel 0.

When a client sends `TRAN_GET_CHAT_HISTORY` with `DATA_CHANNEL_ID = 0`, the server returns history for the public chat.

**Note:** The existing live chat broadcast (`TRAN_CHAT_MSG`, type 106) does not carry a `DATA_CHANNEL_ID` field — it implicitly targets channel 0. If a future specification introduces named channels, the broadcast path will need to include `DATA_CHANNEL_ID` so that capable clients can associate incoming messages with the correct channel.

### Named Channels (Future)

Channel IDs 1 and above are **reserved for a future specification** that defines persistent named channels (analogous to Discord's `#channel-name` concept). This version of the specification does not define how named channels are created, listed, joined, or managed.

Servers MUST reply with an error if a client requests history for a channel ID the server does not recognise.

The reserved transaction types 701–702 and field IDs `0x0F09`–`0x0F1F` are intended for channel management. A future specification will define:

- Channel creation, deletion, and listing transactions
- Channel metadata fields (name, topic, creation date)
- Per-channel access permissions
- Channel membership (if channels are not open to all users)

### Private Chat Privacy

Ephemeral private chats (those created via `TRAN_INVITE_NEW_CHAT` and identified by `DATA_CHATID` / `0x0072`) carry an expectation of privacy. When a user creates a private chat, they are pulling others aside for a transient conversation — like a meeting room with a whiteboard that gets erased when everyone leaves.

**Servers MUST NOT persist chat history for ephemeral private chats.** This is a deliberate design constraint, not an implementation shortcut. Server-side logging of private conversations would violate the trust model that Hotline users expect.

Clients MAY store private chat history locally at the user's discretion. This is a client-side feature and does not involve the server.

---

## Access Privileges

| Bit | Name | Description |
|---|---|---|
| 56 | `accessReadChatHistory` | User may request chat history via Get Chat History (700) |

Bit 56 is the next available bit after `accessVoiceChat` (bit 55).

**Behaviour:**

- Servers SHOULD check bit 56 when processing `TRAN_GET_CHAT_HISTORY`. If the bit is not set, the server replies with an error.
- If the server's access system does not assign semantic meaning to bit 56 (e.g., older server implementations or account schemas that predate this extension), the server SHOULD fall back to checking `ACCESS_READ_CHAT` (bit 9) — the standard chat read permission.
- The fallback ensures that simple implementations can reuse the existing permission, while servers with granular access control can differentiate "can read live chat" from "can read history."
- The `CAPABILITY_CHAT_HISTORY` bit (bit 4 in `DATA_CAPABILITIES`) is still echoed in the login reply regardless of the user's privilege — the capability indicates server support, not user permission. This allows clients to display a history UI in a disabled state (e.g., "Chat history requires permission") rather than hiding it entirely.

---

## Legacy Broadcast Behaviour

Servers MAY send recent chat messages as standard `TRAN_CHAT_MSG` (106) transactions to clients that do not advertise `CAPABILITY_CHAT_HISTORY` (bit 4). This provides a basic "catch-up" experience for older clients without requiring any protocol changes on their side.

The following details are entirely **implementation-defined**:

- Whether to send the broadcast at all
- How many messages to include (a common default is 30)
- When to send them (e.g., after the client requests the user list, indicating it's "ready")
- How to format them (whether to embed timestamps in the message text, whether to include delimiters like "--- Chat History ---")

Servers SHOULD NOT send legacy broadcasts to clients that have negotiated `CAPABILITY_CHAT_HISTORY` — those clients will use `TRAN_GET_CHAT_HISTORY` (700) to retrieve history at their own pace.

---

## Storage

### Requirements

This specification deliberately does not prescribe a storage backend. The server's only obligations are:

1. **Assign monotonically increasing uint64 message IDs.** The generation scheme is implementation-defined.
2. **Support cursor-based queries.** Given a cursor (before/after) and a limit, return the matching messages in ascending ID order.
3. **Enforce retention policy.** Remove messages that exceed the configured maximum count or age.

Beyond these, the server may use any technology: SQLite, JSONL flat files, a custom binary log, an in-memory ring buffer, PostgreSQL, or anything else. This flexibility is the key adoption enabler — a server implementer can start with the simplest approach that works for their platform and scale.

### Encryption at Rest

Servers are encouraged to support encryption at rest for chat history, but the mechanism is implementation-defined. Options include:

- **Database-level encryption** (e.g., SQLCipher for SQLite)
- **File-level encryption** (e.g., encrypting each JSONL line or the entire file)
- **Filesystem-level encryption** (e.g., LUKS, FileVault)
- **Application-level encryption** using existing cryptographic primitives (e.g., ChaCha20-Poly1305, which is already present in servers that implement the HOPE AEAD extension)

The encryption key or passphrase is provided via server configuration and is never transmitted over the wire.

---

## Reserved Transaction Types

The following transaction type IDs are reserved for future extensions to the chat history system. They are not defined in this version of the specification and MUST NOT be used by implementations.

| ID | Name | Purpose |
|---|---|---|
| 701 | *Get Chat History Info* | Query channel listing, metadata, or retention details |
| 702 | *Channel Management* | Create, delete, or modify named channels |
| 703 | *Delete Chat History Message* | Administrator removal of a message (sets tombstone flag) |
| 704 | *Edit Chat History Message* | Administrator or user editing of a message |
| 705–709 | *Unassigned* | Future chat history extensions |

---

## Implementation Notes

- **Message recording hook.** The simplest integration point is in the existing `handle_chat_send` handler. After the server formats the message for broadcast, it passes the same data to the history storage layer. This ensures the persisted message matches exactly what was broadcast.
- **Retention enforcement.** Servers should prune expired messages periodically (e.g., on a timer or after every N inserts), not on every read request. Pruning on reads adds latency to client-facing queries.
- **Default limit.** When `DATA_HISTORY_LIMIT` is absent from a request, the recommended default is 50 messages. Servers MAY enforce a maximum limit (e.g., 200) to prevent excessively large responses.
- **Empty nick and message.** Both `nick_len` and `msg_len` may be zero — this is expected for tombstoned entries (flag bit 2). Parsers MUST handle zero-length strings.
- **Sub-field forward compatibility.** Even though no sub-field types are defined in v1, implementations SHOULD write the parsing logic now (skip unknown sub-types using sub-length). This avoids a compatibility gap when v2 sub-fields are introduced.
- **Concurrent writes during pagination.** New messages arriving while a client paginates backward do not affect correctness — new messages have higher IDs than any cursor the client is using. The cursor is stable.
- **Icon ID accuracy.** The icon ID is captured at the time the message is recorded. If a user changes their icon later, historical messages still show the original icon. This matches the behaviour of other messaging platforms.
- **Server messages and broadcasts.** Admin broadcasts (`TRAN_USER_BROADCAST`, type 355) and server messages (`TRAN_SERVER_MSG`, type 104) MAY be recorded with the `is_server_msg` flag (bit 1) set. The `nick` field for server messages SHOULD be empty or set to the server name.
- **Rate limiting.** Servers SHOULD enforce rate limits on `TRAN_GET_CHAT_HISTORY` to prevent abuse. A reasonable default is 10 requests per second per client. Servers MAY respond with an error or silently drop excessive requests. This is implementation-defined and not part of the wire protocol.

---

## Example: Login with Chat History Negotiation

A client logs in with `CAPABILITY_LARGE_FILES` (bit 0) and `CAPABILITY_CHAT_HISTORY` (bit 4). The server confirms both and includes retention policy.

**Client → Server: Login (107)**

```
Field: DATA_CAPABILITIES (0x01F0)
Length: 2
Value:  00 11    ← bits 0 and 4 set
```

**Server → Client: Login Reply**

```
Field: DATA_CAPABILITIES (0x01F0)
Length: 2
Value:  00 11    ← bits 0 and 4 confirmed

Field: DATA_HISTORY_MAX_MSGS (0x0F07)
Length: 4
Value:  00 00 27 10    ← 10000 messages

Field: DATA_HISTORY_MAX_DAYS (0x0F08)
Length: 4
Value:  00 00 00 1E    ← 30 days
```

Chat history is now active. The client knows the server keeps up to 10,000 messages or 30 days, whichever limit is hit first.

---

## Example: Get Chat History Request and Reply

The client requests the 50 most recent public chat messages (no cursors).

**Client → Server: Get Chat History (700)**

```
 Offset  Bytes                         Field
 ──────  ────────────────────────────  ─────────────────────────────────────
 HEADER (20 bytes)
 00      00                            Flags       = 0
 01      00                            IsReply     = 0 (request)
 02      02 BC                         Type        = 700 (TRAN_GET_CHAT_HISTORY)
 04      00 00 00 42                   ID          = 0x00000042 (task ID)
 08      00 00 00 00                   ErrorCode   = 0
 0C      00 00 00 10                   TotalSize   = 16 (ParamCount + fields)
 10      00 00 00 10                   DataSize    = 16

 PAYLOAD (16 bytes)
 14      00 02                         ParamCount  = 2

         ── Field 1: DATA_CHANNEL_ID (0x0F01) ─────────────────────────────
 16      0F 01                         ID          = DATA_CHANNEL_ID
 18      00 04                         Length      = 4
 1A      00 00 00 00                   Data        = 0 (public chat)

         ── Field 2: DATA_HISTORY_LIMIT (0x0F04) ─────────────────────────
 1E      0F 04                         ID          = DATA_HISTORY_LIMIT
 20      00 02                         Length      = 2
 22      00 32                         Data        = 50
```

Total packet: 20 (header) + 16 (payload) = **36 bytes**.

**Server → Client: Reply**

The server returns 2 messages (abbreviated for clarity) and indicates more are available.

```
 Offset  Bytes                         Field
 ──────  ────────────────────────────  ─────────────────────────────────────
 HEADER (20 bytes)
 00      00                            Flags       = 0
 01      01                            IsReply     = 1 (reply)
 02      02 BC                         Type        = 700 (TRAN_GET_CHAT_HISTORY)
 04      00 00 00 42                   ID          = 0x00000042 (echoed from request)
 08      00 00 00 00                   ErrorCode   = 0
 0C      00 00 00 60                   TotalSize   = 96 (ParamCount + fields)
 10      00 00 00 60                   DataSize    = 96

 PAYLOAD (96 bytes)
 14      00 03                         ParamCount  = 3
         ── Field 1: DATA_HISTORY_ENTRY (0x0F05) ─────────────────────────
         ID: 0F 05
         Length: 00 2A (42 bytes)
         Data:
           00 00 00 00 00 00 03 E8     message_id  = 1000
           00 00 00 00 67 0E 4B 80     timestamp   = 1729137536 (2024-10-17T12:32:16Z)
           00 00                        flags       = 0 (normal message)
           00 80                        icon_id     = 128
           00 04                        nick_len    = 4
           67 72 65 67                  nick        = "greg"
           00 0E                        msg_len     = 14
           48 65 6C 6C 6F 20 65 76     message     = "Hello everyone"
           65 72 79 6F 6E 65

         ── Field 2: DATA_HISTORY_ENTRY (0x0F05) ─────────────────────────
         ID: 0F 05
         Length: 00 27 (39 bytes)
         Data:
           00 00 00 00 00 00 03 E9     message_id  = 1001
           00 00 00 00 67 0E 4B 85     timestamp   = 1729137541
           00 00                        flags       = 0
           00 80                        icon_id     = 128
           00 05                        nick_len    = 5
           61 6C 69 63 65              nick        = "alice"
           00 0A                        msg_len     = 10
           57 68 61 74 27 73 20 75     message     = "What's up?"
           70 3F

         ── Field 3: DATA_HISTORY_HAS_MORE (0x0F06) ─────────────────────
         ID: 0F 06
         Length: 00 01
         Data: 01                       has_more    = 1 (more messages available)
```

---

## Example: Scrollback Pagination

After receiving the messages above, the client scrolls back to load older messages. It uses the oldest message ID (1000) as the `BEFORE` cursor.

```
Request:
  DATA_CHANNEL_ID      = 0
  DATA_HISTORY_BEFORE  = 1000    ← "give me messages before this one"
  DATA_HISTORY_LIMIT   = 50

Reply:
  DATA_HISTORY_ENTRY   (message_id=950, ...)
  DATA_HISTORY_ENTRY   (message_id=951, ...)
  ...
  DATA_HISTORY_ENTRY   (message_id=999, ...)
  DATA_HISTORY_HAS_MORE = 1

Next request (continue scrolling):
  DATA_HISTORY_BEFORE  = 950     ← oldest ID from previous batch
  DATA_HISTORY_LIMIT   = 50

And so on, until:
  DATA_HISTORY_HAS_MORE = 0      ← "you've reached the beginning"
```

The client displays a "beginning of chat history" marker when `has_more` is `0`.

---

## Example: Catching Up After Reconnect

A client reconnects after being offline. Its local cache contains messages up to ID 5000. It uses the `AFTER` cursor to fetch everything it missed.

```
Request 1:
  DATA_CHANNEL_ID     = 0
  DATA_HISTORY_AFTER  = 5000     ← "give me everything after this"
  DATA_HISTORY_LIMIT  = 50

Reply:
  DATA_HISTORY_ENTRY  (message_id=5001, ...)
  DATA_HISTORY_ENTRY  (message_id=5002, ...)
  ...
  DATA_HISTORY_ENTRY  (message_id=5050, ...)
  DATA_HISTORY_HAS_MORE = 1       ← more to catch up on

Request 2:
  DATA_HISTORY_AFTER  = 5050
  DATA_HISTORY_LIMIT  = 50

Reply:
  DATA_HISTORY_ENTRY  (message_id=5051, ...)
  ...
  DATA_HISTORY_ENTRY  (message_id=5073, ...)
  DATA_HISTORY_HAS_MORE = 0       ← caught up to present

The client merges these messages into its local view seamlessly.
```

---

Author: [Greg Gant](https://github.com/fuzzywalrus)  
Status: draft; subject to refinement.
