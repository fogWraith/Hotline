# Draft: Server Emoji

> **Status:** Draft proposal by Misha Nasledov (hxd-ng). **Not adopted**: nothing here is part of the protocol unless another document says so. It proposes capability bit 20, transactions 713–716, field IDs `0x0F38`–`0x0F42`, history sub-field type `0x0022`, and reason codes 7–11, all within the blocks the [Structured Chat](Capabilities-Structured-Chat.md) draft proposes be kept for this family. It also asks for an access bit, Manage Emoji, from the extension allocation, without proposing its number. All of these were unassigned at the time of writing; none is allocated. Until they are, they are used only between servers and clients whose operators and users have chosen to try it, and nothing here changes what is asked of a server or client that has not.

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This extension lets a server keep a small catalog of custom emoji, curated by people its operator trusts and shown to every capable client on that server. They are usable in text as `:name:`, which the sender's client marks as an emoji, and, where the [Reactions](Capabilities-Reactions.md) draft is also offered, as reactions.

It stands alone beside the [Structured Chat](Capabilities-Structured-Chat.md) draft, using the raw text of its structured line where a client has it, and shares the [Replies](Capabilities-Replies.md) draft's reason field and reply shape, using them by reference. Images go through the pipeline of the [Inline Media Extension](Capabilities-Inline-Media.md). For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md).

## Table of Contents

- [What Does Not Change](#what-does-not-change)
- [Negotiation](#negotiation)
  - [Capability Bit](#capability-bit)
  - [Access Bit](#access-bit)
  - [Server Configuration](#server-configuration)
  - [Login Reply](#login-reply)
- [The Catalog](#the-catalog)
- [Names](#names)
- [Images](#images)
- [Get Emoji Catalog (713)](#get-emoji-catalog-713)
- [Get Emoji (714)](#get-emoji-714)
- [Emoji Catalog Changed (715)](#emoji-catalog-changed-715)
- [Manage Emoji (716)](#manage-emoji-716)
- [Server Emoji in Text](#server-emoji-in-text)
- [Reactions](#reactions)
- [Who May Fetch](#who-may-fetch)
- [Text Encoding](#text-encoding)
- [Across Server Links](#across-server-links)
- [Data Objects](#data-objects)
- [Reason Codes](#reason-codes)
- [Performance Considerations](#performance-considerations)
- [Security Considerations](#security-considerations)
- [Hotline-ng Binding](#hotline-ng-binding)
- [Additions to the Tables](#additions-to-the-tables)
- [Open Questions](#open-questions)

---

## What Does Not Change

- **A client that does not negotiate bit 20 gets the text as typed.** A server MUST NOT synthesize an inline image for `:name:`, or anything else, for it: `:name:` reaches it as those characters.
- **The text is never changed.** What makes `:name:` an emoji is a field the sender's client sends beside it ([Server Emoji in Text](#server-emoji-in-text)).

---

## Negotiation

### Capability Bit

| Bit | Mask | Name | Description |
|---|---|---|---|
| 20 | `0x00100000` | `CAPABILITY_SERVER_EMOJI` | The client reads the server's emoji catalog (713–715), sends and understands `DATA_EMOJI_REFS`, and, with bit 19, reactions keyed by a server emoji |

Bit 20 follows the [Replies](Capabilities-Replies.md#capability-bit) draft's rules for its bit, except that it needs bit 1 alone, not bit 17: only beside bit 1, never on a server link, a statement of what the server can, sent in four bytes. It is meaningful without bits 18 and 19: emoji in text. A server that offers bit 20 without them sends no private message references; those belong to bits 18 and 19. A server MUST drop `DATA_EMOJI_REFS` from a sender that did not negotiate bit 20, and relay the rest.

### Access Bit

One access bit is requested from the extension allocation, beside the [Reactions](Capabilities-Reactions.md#access-bit) draft's `AccessReact` and under the same terms: no number named, granted as any bit, bit 5 to edit it if it lands at 64 or above, and a clear bit meaning "may not".

| Name | Allows |
|---|---|
| `AccessManageEmoji` | Adding, renaming and removing server emoji (716) |

### Server Configuration

Recommended settings (implementation-defined):

| Setting | Type | Default | Description |
|---|---|---|---|
| `Emoji.Enabled` | bool | `false` | Keep a server emoji catalog |
| `Emoji.MaxCount` | int | `200` | Live emoji in the catalog |
| `Emoji.MaxBytes` | int | `16384` | Largest stored emoji image, never above 65535. A server that wants long animations may raise it toward 65535 |
| `Emoji.MaxDimension` | int | `128` | Emoji are fitted within this square; never below 64 |
| `Emoji.MaxFrames` | int | `50` | Frames in an animated emoji; at least 16 |

### Login Reply

When bit 20 is confirmed, the login reply MUST carry `DATA_EMOJI_REVISION`, the catalog's current revision, and SHOULD carry `DATA_MAX_EMOJI`, `DATA_MAX_EMOJI_BYTES` and `DATA_MAX_EMOJI_DIMENSION`.

---

## The Catalog

A server's emoji are a list of entries, each:

| Property | Meaning |
|---|---|
| ID | A UInt32 the server assigns, **never reused**, never changed. What reactions and refs name |
| Name | The shortcode, `[a-z0-9_]{2,32}`. Unique among live entries. May be changed |
| Hash | The SHA-256 of the stored image. Changes only with the image |
| Width, height | Of the stored image, in pixels |
| Animated | The stored image is an animated GIF |
| Removed | The entry was removed; only its ID and last name remain |

The catalog has a **revision**, a UInt32 the server increments on every change, and each entry records the revision it last changed at. A client that holds the catalog at a revision need not fetch it again until it learns of a later one, and then fetches only what changed.

**An ID, not a name or a hash.** A name moves: keyed by name, a reaction would follow a rename to whatever emoji takes the name next. A hash is one image's: replacing a poor drawing would orphan every reaction made with it. The ID is the identity, the name a label, the hash a cache key. An image is never changed in place; a new image is a new entry, so an ID always draws the same picture.

A removed entry keeps its ID and last name, so a reaction made with it still says `:name:`. A server keeps up to `Emoji.MaxCount` removed entries, forgetting the longest removed first, and never scans messages or reactions to decide.

---

## Names

- `[a-z0-9_]`, 2 to 32 characters. No uppercase, so `:Party:` and `:party:` are never two emoji; no other punctuation, so a name is always safe to show as text and cannot carry a URL, markup or a bidirectional control.
- A server SHOULD refuse a name that is a common Unicode emoji shortcode (the [gemoji](https://github.com/github/gemoji) names, such as `:joy:` and `:thumbsup:`). Clients already turn those into Unicode emoji locally, and a server emoji shadowing one would draw differently on different clients.
- A name freed by a removal or a rename may be taken by another entry, which gets a new ID.

---

## Images

Every stored emoji image is one the server **decoded and re-encoded itself**, through the pipeline the [Inline Media Extension](Capabilities-Inline-Media.md#upload-validation-pipeline) requires of a media upload. That pipeline is a magic-byte sniff, a structural walk with no trailing bytes, dimension and pixel probes before any allocation, a bounded decode, and a re-encode that writes no metadata. The image is then **fitted** within `Emoji.MaxDimension` square, keeping its aspect ratio.

- Formats: PNG, GIF and JPEG in; PNG out for a still, GIF out for an animation. **No SVG**, ever, and no format the server cannot re-encode.
- An animation is bounded by `Emoji.MaxFrames` and 10 seconds in total, before any frame is decoded, and frames times pixels is bounded against the same allocation budget a still gets.
- The stored image MUST fit `Emoji.MaxBytes`, at most 65535, so it travels in one field. An animation that does not fit after fitting MAY be stored as its first frame, or refused `InvalidImage`.
- Width, height and hash are measured from the stored bytes.

---

## Get Emoji Catalog (713)

**Client → Server (request/reply).** Requires bit 20.

| Field | Type | Required | Description |
|---|---|---|---|
| `DATA_EMOJI_REVISION` | UInt32 | No | The revision the client holds |
| `DATA_EMOJI_ID` | UInt32 | No | Continue a listing after this ID |

**Reply fields:** `DATA_EMOJI_REVISION`, the current revision. A request with the current revision gets nothing more. One with an older revision that the server can still answer from gets the entries changed since it, live and removed. One with none, or older than that, gets every entry the server keeps, live and removed. Entries go in ascending ID order, starting after `DATA_EMOJI_ID` when the request carries it.

**Paging.** A server MUST keep the reply within the transaction size clients accept. When the entries do not all fit, it sends the lowest IDs that fit and `DATA_EMOJI_MORE` = `1`, and the client asks again with the same revision and `DATA_EMOJI_ID` set to the last ID it received.

`DATA_EMOJI_ENTRY` is packed: `u32 id`, `u32 revision` (when it last changed), `u16 flags` (bit 0 animated, bit 1 removed), `u16 width`, `u16 height`, `16 bytes hash` (the first 16 bytes of the SHA-256, as `DATA_BUDDY_ICON_HASH` is; zero for a removed entry), `u8 name_len`, `name`. An entry is at most 63 bytes. A removed entry the server no longer keeps is drawn as the name the message's refs carry, or as an empty placeholder in a reaction.

A server SHOULD ration Get Emoji Catalog per session.

---

## Get Emoji (714)

**Client → Server (request/reply).** Requires bit 20.

**Request fields:** `DATA_EMOJI_ID`. **Reply fields:** `DATA_EMOJI_ID`, `DATA_EMOJI_IMAGE`, the stored bytes. A removed or unknown ID is a task error. Since an ID always names the same bytes, a client MAY cache an image indefinitely under its ID, and MUST drop it once the catalog marks the entry removed.

A server SHOULD rate-limit Get Emoji per session, with a burst large enough that a client opening a busy chat fetches every emoji it needs at once. A client SHOULD fetch images lazily, when it first draws them, with a little random delay, and keep at most four 714s outstanding.

---

## Emoji Catalog Changed (715)

**Server → Client (notification).** Sent to every session that negotiated bit 20 when the catalog changes, carrying only `DATA_EMOJI_REVISION`. Delivery is pull-based, as in [GIF Icons](GIF-Icons.md). A server SHOULD coalesce changes into at most one notification every few seconds (5 s is a reasonable minimum), so an operator's batch of edits costs every session one fetch.

---

## Manage Emoji (716)

**Client → Server (request/reply).** Requires bit 20 and `AccessManageEmoji`.

| `DATA_EMOJI_ACTION` | Fields | Does |
|---|---|---|
| `1` add | `DATA_EMOJI_NAME`, `DATA_EMOJI_IMAGE` | A new entry, under a new ID |
| `2` rename | `DATA_EMOJI_ID`, `DATA_EMOJI_NAME` | Changes the name; ID, image and every use stay |
| `3` remove | `DATA_EMOJI_ID` | Marks the entry removed |

**Reply fields:** on success, the resulting `DATA_EMOJI_ENTRY` and the new `DATA_EMOJI_REVISION`. Refusals use the [reply shape](Capabilities-Replies.md#reply-shape) with `DATA_CHAT_REASON`: `NotAllowed` (before the image is decoded), `InvalidName`, `NameTaken`, `CatalogFull` (at `Emoji.MaxCount` live entries), `InvalidImage` (with no more said), `NoSuchEmoji`, `RateLimited`.

Adding an emoji puts an image on every capable screen on the server without anyone asking, so it is a privilege the operator grants. A server SHOULD rate-limit every action per account, adds most tightly, since each costs a decode.

**Add is optional.** A server MAY provision its catalog with an operator tool that runs the same pipeline offline, and answer an add `NotAllowed`. Rename and remove are cheap and SHOULD be offered.

**Removal.** A removed emoji is no longer offered, fetched or accepted in a new reaction or ref. What already named it, reactions included, keeps naming its ID, and is drawn as the entry's last `:name:`, as text.

---

## Server Emoji in Text

A user types a server emoji as `:name:`, as in Slack and Discord. The text is sent, stored and relayed **exactly as typed**, and the server never reads it. What makes `:name:` an emoji is `DATA_EMOJI_REFS`, which the **sender's client** attaches, binding names to IDs for that one message.

- A client that negotiated bit 20 MAY add `DATA_EMOJI_REFS` to Send Chat (105), Send Instant Msg (108) and IM Send (810), listing each server emoji its text uses, once, with its ID.
- **The server checks refs against its catalog, never against the text.** It keeps an entry only if its ID names a live entry whose current name is exactly the entry's name, and silently drops every other entry, any duplicate, and any past the first 32. It does not look for the names in the text: a ref the text never uses is kept, and readers ignore it. A message is never refused over its refs.
- The server keeps the refs wherever it keeps the message, and relays them to sessions that negotiated bit 20: as `DATA_EMOJI_REFS` beside the text on Chat Message (106), Server Message (104) and IM Deliver (811), and in Get Chat History (700) entries as sub-field `0x0022` `EMOJI_REFS` ([Chat History](Capabilities-Structured-Chat.md#chat-history)). Every other session receives the text, which reads `:party:`.
- A capable client draws a `:name:` token as an emoji only when the message's refs bind that name; every other `:name:` is text, or a Unicode shortcode as the client already handles it. A later rename leaves the binding by ID: the image is the one the sender saw, and the text still says what they typed.
- **Where to look for tokens on 106.** A client that also negotiated bit 17 SHOULD match `:name:` tokens against its [structured line](Capabilities-Structured-Chat.md)'s raw text rather than the formatted line, where a nick containing `:name:` could be mistaken for one. A client without it looks for tokens after each segment's nick prefix in the formatted line, and accepts that the prefix is ambiguous: servers format lines differently, emotes have no colon, and a reply may be decorated. Only names the message's refs bind are drawn as emoji, which limits the harm of a misread to a name the sender meant anyway.
- A client that sends no refs gets no emoji. A period client's `:party:` stays text for every reader.
- `DATA_EMOJI_REFS` is packed: `u16 n`, then `n` entries of `u32 id`, `u8 name_len`, `name`.

**Why not a token in the text.** `<:party:1234>` is unreadable on every client that does not know it, which is every classic client. It also lets a sender write a name the image does not have (`<:lol:1234>` where 1234 is something else), so that different readers see different things. Refs keep the text readable, and the server's check pins each name to its ID.

**Signed and sealed messages.** The refs of a signed message are not part of what it signs, as the nick is not. A sealed message has no refs the server can see; whether they go inside the sealed body, checked by the recipient against the catalog, is asked in the [Replies](Capabilities-Replies.md#open-questions) draft.

---

## Reactions

Where bit 19 is also confirmed, a [reaction key](Capabilities-Reactions.md#reaction-keys) of kind 2 names a catalog entry by its ID. It names an ID, never a name: a rename changes nothing about the reactions already made with it, and a name freed by a removal and taken by a new emoji does not inherit them. A kind 2 key from a session that did not negotiate bit 20 is refused `InvalidKey`, and one naming no live entry `NoSuchEmoji`. A removed emoji stays in the reactions that use it, drawn as its `:name:` text; no new reaction may use it.

---

## Who May Fetch

A server emoji is shown to the whole server by design, like a user list icon. Any session on the server MAY fetch any live emoji, and nothing about who was shown it is recorded. This is the opposite of an inline media handle, which is an authorization fixed when the message was relayed and checked on every fetch. The difference is safe because of where each comes from: a media image is any user's upload, aimed at one audience, while an emoji is curated by someone the operator trusts and is the same for everyone. A server MUST NOT promote an inline media handle into the catalog except by an add from someone holding `AccessManageEmoji`, and MUST NOT accept a media handle where an emoji ID is expected.

---

## Text Encoding

Bit 20 requires `CAPABILITY_TEXT_ENCODING` (bit 1). Emoji refs are **Binary** fields, never transcoded. Emoji names are ASCII, so a `:name:` typed by a Mac Roman client survives its encoding; that client cannot negotiate bit 20 and sends no refs, so its `:name:` stays text.

---

## Across Server Links

Server emoji are one server's. Refs do not cross a link, and a server attaches none to a linked user's line, so a peer's `:party:` is never drawn as this server's. A reader on a peer sees the shortcode as text. No server fetches an emoji from another server on a reader's behalf, and no client fetches one from anywhere but the server it is connected to.

---

## Data Objects

Within the family's block ([Structured Chat](Capabilities-Structured-Chat.md#data-objects)). `DATA_CHAT_REASON` is the [Replies](Capabilities-Replies.md#data-objects) draft's. Integers are big-endian.

| ID (hex) | Decimal | Name | Type | Description |
|---|---|---|---|---|
| `0x0F38` | 3896 | `DATA_EMOJI_ID` | UInt32 | A catalog entry's ID |
| `0x0F39` | 3897 | `DATA_EMOJI_NAME` | String | A name, `[a-z0-9_]{2,32}` |
| `0x0F3A` | 3898 | `DATA_EMOJI_IMAGE` | Binary (at most 65535) | An emoji image |
| `0x0F3B` | 3899 | `DATA_EMOJI_ENTRY` | Binary | One packed catalog entry |
| `0x0F3C` | 3900 | `DATA_EMOJI_REVISION` | UInt32 | The catalog's revision |
| `0x0F3D` | 3901 | `DATA_EMOJI_ACTION` | UInt16 | 1 add, 2 rename, 3 remove |
| `0x0F3E` | 3902 | `DATA_EMOJI_REFS` | Binary | Names bound to IDs for one message |
| `0x0F3F` | 3903 | `DATA_EMOJI_MORE` | UInt8 | `1` when a Get Emoji Catalog reply has more entries after its last |
| `0x0F40` | 3904 | `DATA_MAX_EMOJI` | UInt32 | Login reply: `Emoji.MaxCount` |
| `0x0F41` | 3905 | `DATA_MAX_EMOJI_BYTES` | UInt32 | Login reply: `Emoji.MaxBytes` |
| `0x0F42` | 3906 | `DATA_MAX_EMOJI_DIMENSION` | UInt32 | Login reply: `Emoji.MaxDimension` |

---

## Reason Codes

The family's enumeration is the [Replies](Capabilities-Replies.md#reason-codes) draft's. This draft adds:

| Code | Name | Meaning |
|---|---|---|
| 7 | `NoSuchEmoji` | No live catalog entry has that ID |
| 8 | `NameTaken` | A live entry has that name |
| 9 | `InvalidName` | A name outside [Names](#names)' rules |
| 10 | `InvalidImage` | The image was refused |
| 11 | `CatalogFull` | The catalog is at `MaxCount` |

---

## Performance Considerations

- **The catalog.** The login reply's revision lets a client that holds the catalog skip 713, and 713 answers with what changed since the client's revision.
- **Images** are immutable by ID and by hash and cacheable indefinitely; the ng route sends `immutable` cache headers. Clients fetch them lazily and a few at a time.
- **Churn.** Catalog notifications coalesce over seconds, and every 716 action is rate-limited.
- **Small servers.** Adding through 716 is optional; a catalog may be provisioned offline, and removed entries are kept by count, never by scanning what refers to them.

---

## Security Considerations

- **No image arrives unasked from a stranger.** Only holders of `AccessManageEmoji` add to the catalog, and only the catalog's images appear in reactions and refs.
- **Images are the server's own encodings.** No SVG, no metadata, no polyglots, a bounded decode. A client fetches emoji only from the server it is connected to, and MUST NOT fetch one from a URL, a peer server, or anywhere a message names.
- **Names are inert**, and **refs cannot be spoofed**: the server keeps a ref only when its ID and name match a live entry.
- **Bounded input.** Refs are at most 32 entries.

---

## Hotline-ng Binding

*Implementers of the classic extension can skip to [Additions to the Tables](#additions-to-the-tables).*

For a server that also speaks the [Hotline-ng](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng.md) JSON protocol, the same feature as capability `emoji`, under the same rules. The login block `emoji`: `{ "revision", "max_count", "max_bytes", "max_dimension", "manage": bool }`.

| `req` | params | ok | errors | weight |
|---|---|---|---|---|
| `emoji_list` | `revision?`, `after?` (an ID) | `{ "revision", "emoji"?: [ { "id", "revision", "name", "hash", "width", "height", "animated"?, "removed"? } ], "delta"?: true, "has_more"?: true }`: only what changed since `revision` when `delta` is set, in ascending ID order, continued with `after` | `not_available` | a read |
| `emoji_rename` | `id`, `name` | `{ "emoji", "revision" }` | `access_denied`, `no_such_emoji`, `name_taken`, `bad_request` (an invalid name), `rate_limited` | a write |
| `emoji_remove` | `id` | `{ "revision" }` | `access_denied`, `no_such_emoji`, `rate_limited` | a write |

Event `emoji_changed`: `{ "revision" }`. `hash` is the lowercase hex of the full SHA-256, as an avatar's `id` is. Bytes over HTTP, as a server's other images are:

```
GET /emoji/{hash}               Authorization: Bearer <session>.<token>
  200  image/png | image/gif, Cache-Control: private, max-age=31536000, immutable
  404  no live emoji has that hash

POST /emoji?name={name}         Authorization: Bearer <session>.<token>
  201 { "emoji": { … }, "revision": … }
  403 no AccessManageEmoji, 409 name taken, 413, 415, 422 invalid name or catalog full, 429
```

An add is a heavy request, as an avatar upload is. `chat` and `msg` take `emoji: [ { "name", "id" } ]` from the sender's client, checked as on the classic wire. The `chat`, `msg` and `im` events, `history` and `inbox` rows, and the messages of a `requests` listing carry what was kept. A reaction key `{ "custom": id }` is the [Reactions](Capabilities-Reactions.md#hotline-ng-binding) binding's.

---

## Additions to the Tables

All **proposed, pending allocation**, within the blocks the [Structured Chat](Capabilities-Structured-Chat.md#additions-to-the-tables) draft proposes be kept for this family.

Capability bit, as a row of [Capabilities.md](Capabilities.md#defined-capability-bits):

| Bit | Mask | Name | Description |
|---|---|---|---|
| 20 | `0x00100000` | `CAPABILITY_SERVER_EMOJI` | **Draft**, not adopted. A server's own emoji: a catalog (713–716), `DATA_EMOJI_REFS` marking `:name:` in text, and, with bit 19, reaction keys. Images are re-encoded by the server and fetched only from it. Only alongside bit 1, and never on a server link (bit 11). |

Transactions:

| ID | Name | Type | Description |
|---|---|---|---|
| 713 (`0x02C9`) | Get Emoji Catalog | Request/reply | The catalog, or what changed since a revision |
| 714 (`0x02CA`) | Get Emoji | Request/reply | One emoji's image, by ID |
| 715 (`0x02CB`) | Emoji Catalog Changed | Server → client | The catalog's new revision |
| 716 (`0x02CC`) | Manage Emoji | Request/reply | Add, rename or remove an emoji |

Fields: `0x0F38`–`0x0F42`, as in [Data Objects](#data-objects).

Chat history sub-field `0x0022` `EMOJI_REFS`. Reason codes 7–11, in the family's enumeration.

Access bit: `AccessManageEmoji`, **requested**. Bits 55–60 are taken, and no number is named here.

---

## Open Questions

1. **The access bit.** Which number would you give `AccessManageEmoji`?
