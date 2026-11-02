# Draft: Replies

> **Status:** Draft proposal by Misha Nasledov (hxd-ng). **Not adopted**: nothing here is part of the protocol unless another document says so. It proposes capability bit 18, transactions 717 and 718, field IDs `0x0F20`–`0x0F24`, `0x0F2A`, `0x0F2E`–`0x0F30` and `0x0F32`, history sub-field type `0x0020`, repeated `DATA_LINE_ID` in Get Chat History (700) requests, reason codes 0–2, 6, 12 and 13, and notification type 0. All of them lie within the blocks the [Structured Chat](Capabilities-Structured-Chat.md) draft proposes be kept for this family, and were unassigned at the time of writing; none is allocated. Until they are, they are used only between servers and clients whose operators and users have chosen to try it, and nothing here changes what is asked of a server or client that has not.

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This extension lets a line of chat or a private message say which message it answers. The server stores and relays the reference as the sender gave it, and never resolves it for a capable reader. On the classic wire it names a chat line by the ID the [Structured Chat](Capabilities-Structured-Chat.md) draft gives it, and gives a private message a reference of its own. An author may ask to be told when someone answers them.

It builds on [Structured Chat](Capabilities-Structured-Chat.md), which carries a chat line as data on Chat Message (106), and defines the targets, notifications, reason field and reply shape that its companions use. The [Reactions](Capabilities-Reactions.md) draft attaches emoji to the same targets, and the [Server Emoji](Capabilities-Server-Emoji.md) draft adds a server's own emoji to text and reactions. Each has its own capability bit, and a server may offer any of them beside Structured Chat, or Server Emoji alone. For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md). Kept lines' IDs are the [Chat History Extension](Capabilities-Chat-History.md)'s, and instant messages are the [Instant Messaging Extension](Capabilities-Messaging.md)'s.

## Table of Contents

- [What Does Not Change](#what-does-not-change)
- [Negotiation](#negotiation)
  - [Capability Bit](#capability-bit)
  - [Server Configuration](#server-configuration)
  - [Login Reply](#login-reply)
  - [Reply Shape](#reply-shape)
- [Targets](#targets)
  - [Line IDs](#line-ids)
  - [Private Message References](#private-message-references)
  - [Packed Target](#packed-target)
- [Replies](#replies)
  - [How Other Readers See a Reply](#how-other-readers-see-a-reply)
- [Chat History](#chat-history)
- [Notifications](#notifications)
  - [Set Notifications (717)](#set-notifications-717)
  - [Notification (718)](#notification-718)
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

- **A client that does not negotiate bit 18 is sent no reply field.** A server MUST NOT synthesize a separate line or any field on its behalf. It MAY decorate the formatted line that client is sent, and nothing else ([How Other Readers See a Reply](#how-other-readers-see-a-reply)).
- **The text the sender typed is never changed, and never read.** It is stored and relayed as typed. A server MAY decorate only the formatted line, or a 104's text, it sends to a session without bit 18; the stored text, the raw text of a [structured line](Capabilities-Structured-Chat.md), what a bit-18 session is sent, and anything signed are never altered. Nothing here has the server parse the text: a reference is a field the sender's client sends beside it.
- **Live chat stays one flat stream.** There are no threads. A reply is a pointer a capable client MAY draw as a short quote or a jump link; it does not move the line anywhere.
- **Private chats are never stored.** Their lines carry [ephemeral IDs](Capabilities-Structured-Chat.md#ephemeral-line-ids) so they can be answered while they are on screen. Those IDs live in the server's memory for the life of the chat and are never written anywhere; the Chat History Extension's rule that private chats are not persisted holds unchanged. A private message that the server does not store is treated the same way.

---

## Negotiation

### Capability Bit

| Bit | Mask | Name | Description |
|---|---|---|---|
| 18 | `0x00040000` | `CAPABILITY_REPLIES` | The client sends and understands `DATA_REPLY_TO`, is sent private message references, and understands Notification (718) |

This is a bit above 15, so a client that sets it sends `DATA_CAPABILITIES` in four bytes, and a server that confirms it answers in four ([Field Width](Capabilities.md#field-width)).

- A server MUST NOT confirm bit 18 unless it also confirms bit 1 (`CAPABILITY_TEXT_ENCODING`) and bit 17 (`CAPABILITY_STRUCTURED_CHAT`), and MUST NOT confirm it on a server link (bit 11). The same holds for the [Reactions](Capabilities-Reactions.md) draft's bit 19. The [Server Emoji](Capabilities-Server-Emoji.md) draft's bit 20 needs bit 1 alone.
- A bit says the server can, not that this user may, as for chat history. A server confirms it regardless of the account's permissions, and refuses a particular act with a reason, so a client can show a disabled control rather than none.
- A server MUST drop `DATA_REPLY_TO` from a sender that did not negotiate bit 18, and relay the rest.

### Server Configuration

Recommended settings (implementation-defined):

| Setting | Type | Default | Description |
|---|---|---|---|
| `Replies.Enabled` | bool | `false` | Store and relay `DATA_REPLY_TO` |
| `Replies.ClassicForm` | `none` \| `nick` \| `quote` | `nick` | How a reply's formatted line is decorated for a session without bit 18 ([How Other Readers See a Reply](#how-other-readers-see-a-reply)) |
| `Replies.ClassicQuoteLength` | int | `40` | The longest excerpt `quote` copies, in characters |
| `Notifications.DefaultTypes` | bitmask | replies | What a session that never [set its notifications](#set-notifications-717) is told of |

### Login Reply

When bit 18 is confirmed, the login reply MUST carry `DATA_REPLY_TARGETS`: the [target kinds](#packed-target) this server keeps and relays in `DATA_REPLY_TO`, as `1 << kind`. A client MUST NOT send a kind the field leaves out, and a server that receives one drops the reference.

### Reply Shape

Set Notifications (717), and the companions' requests, reply in the [Instant Messaging Extension's reply shape](Capabilities-Messaging.md#reply-shape). A failure has a non-zero error code and `FieldError` (100) text written for the user, and SHOULD carry `DATA_CHAT_REASON` ([Reason Codes](#reason-codes)). A success MAY carry `DATA_CHAT_REASON` set to `OK`, and clients MUST NOT require it.

---

## Targets

A reply names its **target**: the message it answers, by an identifier that the person naming it and everyone reading the result already hold. The companions name the same targets the same way.

| Kind | Name | Names | Exists where |
|---|---|---|---|
| 1 | Line | A line of public chat, or of a persistent channel, by its channel ID and its Chat History message ID | Only where the server keeps chat history for that channel, for lines it kept |
| 2 | Instant message | An instant message, by its sender's Login and its `DATA_MESSAGE_GUID` | Only between the two parties of the [Instant Messaging Extension](Capabilities-Messaging.md) |
| 3 | Signed | A signed line or message, by its signer's identity fingerprint, its 16-byte id and the SHA-256 of the whole signed object, signature included (a reader holding a link form computes it once it has fetched the certificate) | Reserved. A classic server MUST drop a kind 3 target until a version of the [messaging identity draft](Capabilities-Messaging-Identity.md) defines signed chat on this wire |
| 4 | Ephemeral line | A line the server did not keep, such as a private chat's, by its chat ID (`0` for public chat) and its [ephemeral ID](Capabilities-Structured-Chat.md#ephemeral-line-ids) | Only while the chat exists, for its recent lines |
| 5 | Private message | A private message (108/104), by the [reference](#private-message-references) the server stamped on it | Only between its sender and its recipient |

- **A line's message ID** is the one identifier of a public line that the server assigns and every reader holds. It is opaque and compared for order and equality only.
- **An instant message's GUID** is generated by its sender and delivered with it in IM Deliver (811). IM Acknowledge (812) already names a message by the GUID and its sender's Login, and this draft names it the same way.
- **A private chat's line and a classic private message** carry no identifier today, and a private chat's line is never stored, so the server gives each an identifier of its own.

### Line IDs

A line target and an ephemeral line target name a line by the ID its [structured line](Capabilities-Structured-Chat.md#line-ids) carries on Chat Message (106): a history ID, named as kind 1 with its channel, or an ephemeral ID (`is_ephemeral` set), named as kind 4 with its chat ID, `0` for public chat. A client learns its own line's ID from its echo ([The Sender's Echo](Capabilities-Structured-Chat.md#the-senders-echo)).

**A persistent channel shown as a private chat.** A server may relay a persistent channel to classic clients as a private chat, under `DATA_CHATID`, its lines carrying their history IDs. A client names such a line as kind 4 with that chat ID, as it names any line it saw under a chat ID. The server MUST accept that kind 4 target as the kind 1 target of the same line, and MUST present targets to each session in the kind it addresses the channel by: kind 4 with the chat ID to a session that sees a private chat, and kind 1 to one that sees the channel. The client never needs to know which chats are channels.

### Private Message References

A server that accepts kind 5 stamps a reference on every classic private message. A message is identified by whether the server **stores** it, not by how it is delivered. A server that keeps a message in a store, to deliver it later or to keep it for the recipient after delivering it at once, gives it a durable reference. A server that relays a message and keeps nothing gives it an ephemeral one.

| Message | Reference | Given to |
|---|---|---|
| Not stored | **Ephemeral**: 16 bytes from a cryptographically secure random source, stored bit clear | The recipient on the 104; the sender in the reply to its 108 |
| Stored | **Durable**: 16 random bytes chosen when the message is stored, kept with it, stored bit set | The recipient on the 104 that delivers it, live or later; the sender in the reply to its 108 |

Each goes only to a session that negotiated bit 18 or bit 19 (the [Reactions](Capabilities-Reactions.md) draft's), as `DATA_PM_ID`, packed as `u8 flags` (bit 0: stored), then the 16-byte reference. The 16 bytes are the same ones the [Hotline-ng binding](#hotline-ng-binding) names the message by, so both wires name one message alike.

**A reference MUST NOT reveal the volume of the server, or of any mailbox.** Both kinds are random. A store's own ordered message number, which may serve a client as a paging cursor, is never a reference. No published extension tells a classic client that it is reading a stored message or what its identifier is, so this draft is the first to put one on the wire ([Open Questions](#open-questions)).

**A reply's reference is never checked for a bit-18 reader against the store**, or against any memory of what was sent; the one lookup is to decorate a recipient's own copy ([How Other Readers See a Reply](#how-other-readers-see-a-reply)). Its reader resolves it from its own mailbox or its own copy, so a reference to another pair's message shows them nothing. Answering by whether a message still exists would let a sender learn whether it was read or deleted. A server needs no memory of the references it has stamped. The [Reactions](Capabilities-Reactions.md#who-reacts-and-as-whom) draft looks a stored reference up for a party to it, and answers the same whether or not it finds the message.

### Packed Target

`DATA_REPLY_TO`, `DATA_TARGET` and `DATA_NOTIFY_SOURCE` carry a target in one packed form, at most 128 bytes. All integers are big-endian.

| Kind | Layout | Size |
|---|---|---|
| 1 Line | `u8 kind = 1`, `u32 channel`, `u64 line_id` | 13 |
| 2 Instant message | `u8 kind = 2`, `16 bytes guid`, `u8 login_len`, `login` (UTF-8) | 18 + login |
| 3 Signed | `u8 kind = 3`, `32 bytes signer`, `16 bytes id`, `32 bytes sha256` (of the whole signed object) | 81 |
| 4 Ephemeral line | `u8 kind = 4`, `u32 chat_id`, `u64 line_id` | 13 |
| 5 Private message | `u8 kind = 5`, then `DATA_PM_ID`'s 17 bytes | 18 |

`channel` is `0` for public chat, and otherwise the channel ID Get Chat History takes for it. A server parses a target, keeps and relays only the parsed target in its own encoding, and never passes on bytes it did not parse. A reader MUST ignore a target of a kind it does not know, or shorter than its kind's layout.

---

## Replies

With bit 18 confirmed, a client MAY add `DATA_REPLY_TO` to:

| Transaction | Target kinds allowed |
|---|---|
| Send Chat (105), public or a persistent channel | 1 (a line in the same channel), or 4 with chat 0 (an ephemeral public line) |
| Send Chat (105) with `DATA_CHATID` | 4 (a line in the same chat) |
| Send Instant Msg (108) | 5 (a message between the same two users) |
| IM Send (810) | 2 (a message in this same conversation) |

The server keeps the reference wherever it keeps the line or message itself, and relays it to every recipient session that negotiated bit 18: on Chat Message (106) as the structured line's `REPLY_TO` sub-field ([Chat History](#chat-history)), in Server Message (104) and IM Deliver (811) as `DATA_REPLY_TO`, and in [Get Chat History](#chat-history). A reply in a private chat, or in a private message the server does not store, is relayed and forgotten, as its text is. Other sessions receive the line or message without it, its formatted text decorated as below.

- **The server never resolves the target for a bit-18 reader.** It does not quote it or tell the reader whether it exists. A client that wants to draw a quote fetches the target through Get Chat History or its own copy, under its own access, so a reply never shows a reader a line the reader could not read. The server looks up a target only to [notify](#notifications) its author, and to decorate the formatted line of a reader without bit 18, under that reader's own access ([How Other Readers See a Reply](#how-other-readers-see-a-reply)).
- **A reply is never refused, or dropped, as dangling.** A target retired by retention, removed by a moderator, before the reader's history, or never seen by the reader is drawn as the reply's text alone. So is an ephemeral target its reader never held, as for a member who joined a private chat after the line it answers. There is no limit on how far back a reply may point.
- **No reaching across conversations.** A line target names the channel the reply is sent in, an ephemeral line target names the chat it is sent in, and an instant-message target names the sender or the recipient as its Login. A server MUST drop a `DATA_REPLY_TO` that breaks this rule, that is malformed or over 128 bytes, or that names line ID `0` ([Lines Sent to One Session](Capabilities-Structured-Chat.md#lines-sent-to-one-session)), and relay the message without it, on every transaction alike: the classic wire has no refusal to give. These checks are structural, comparing the target's own fields with the message's, and never look anything up. A private-message target is always accepted, and its reader resolves it from its own mailbox ([Private Message References](#private-message-references)). Answering by whether the server still holds a target would let a sender learn whether it was read or deleted.
- **The server never adds a hash** to a target that is not signed. A server could rewrite a line and its hash together, so a hash of an unsigned line proves nothing.
- **Signed replies.** Where a reply is itself signed, the reference belongs inside the signed object, so the signature binds the reply to that exact message. How is the messaging identity draft's to define.

### How Other Readers See a Reply

A reply to an old message can read as a non sequitur to a client that cannot draw it, and people in classic chat already write `alice: ` before an answer by hand. So that clients with a reply button do not train them out of a habit every other reader depends on, a server MAY decorate the **formatted** text it sends a session without bit 18: `FieldData` of Chat Message (106), and the text of Server Message (104) for a private message's reply. `Replies.ClassicForm` chooses how:

| Value | The reader sees |
|---|---|
| `none` | The reply as typed, as today |
| `nick` (default) | The IRC convention: the target's author's nick and a colon before the text, as in `\r          bob:  alice: yes, it is quick for me` |
| `quote` | A short excerpt of the target, as in `\r          bob:  [re alice: has anyone tried the new build on a Qua…] yes, it is quick for me` |

- **Only the formatted line.** The decoration goes before the reply's text, inside the line the server formats, never as a line of its own. The text as typed is what the server stores, what a structured line's raw text carries (so a session with bit 17 and not bit 18 finds it undecorated in its entry), what a bit-18 session is sent, what Link Chat (904) carries, and what any signature covers. A multi-line send is decorated on its first segment only, though a server prefixes each segment with the nick.
- **Never an emote.** A line with `is_action` (`\r *** bob waves`) is never decorated: a client that splits a line at its first colon would read `*** bob alice` as the speaker.
- **Never anywhere else.** IM Deliver (811) is never decorated, and neither is Chat History's legacy replay: the replay is history, and decorating it would cost a lookup for every line replayed.
- **The nick** is the one this reader was shown the target under: its author's nick as that line was drawn for them, tagged for a ghost where the tag option is on. For an ephemeral line target, it is the nick the server keeps for the line in its [ephemeral memory](Capabilities-Structured-Chat.md#ephemeral-memory); past the window, or where the server keeps none, the line goes undecorated. Where the server does not know the author, or the target cannot be named to this reader, the line goes undecorated as well. A target cannot be named to a reader when it has ID `0`, when it is an ephemeral line relayed before they joined, or when it is a kept line older than the first line relayed live to this session that the reader's history access does not reach (`AccessReadChatHistory`, 56, falling back to 9), as the [Reactions](Capabilities-Reactions.md#who-reacts-and-as-whom) draft rules for a line target.
- **The excerpt** is at most `Replies.ClassicQuoteLength` characters of a kept line's text, the trailing ellipsis included, cut at a character boundary, on one line, its line breaks replaced by spaces, and converted to the reader's encoding with `?` for what it cannot write. A server quotes only what this reader may read, under the same rule as the nick. An ephemeral line target is never quoted, since its text is not kept, and falls back to `nick`; so does a tombstone, and a target the server cannot resolve, or to nothing where even the nick cannot be shown.
- **Length.** The decoration's nick is converted to the reader's encoding and capped as a nick is, after conversion. The decoration never counts against the sender's caps. Where the decorated line would pass the server's bound on a line, the excerpt is cut first, then the decoration dropped, and the reply's text is never cut for it.
- **An excerpt is a copy.** It is written onto classic screens and into their logs, where a later redaction of the original cannot reach it. That is why `quote` is opt-in.
- **Private chats and private messages** follow the same rules for their parties. A private message's reply is decorated only from a stored message whose sender and recipient are this reply's two parties; any other target, unstored or not, leaves the 104 undecorated. The decoration is shown to the recipient alone and tells the sender nothing, so it is no probe.
- **Who is decorated.** Every session without bit 18 that is sent the reply. The sender itself has bit 18, since a reference from any other is dropped, so its echo is never decorated; another session of its account without bit 18 is a reader like any other.

---

## Chat History

A line's entry, in a Get Chat History (700) reply and as a [structured line](Capabilities-Structured-Chat.md#chat-history) on Chat Message (106) alike, carries sub-field `0x0020` `REPLY_TO`, the line's `DATA_REPLY_TO` as kept, to sessions that negotiated bit 18. It is in the family's range, whose registry, audience rule, tombstone rule and reply-size rule the [Structured Chat](Capabilities-Structured-Chat.md#chat-history) draft gives.

**Lines by ID.** So that a client can draw the quotes of a page's replies in one request, Get Chat History (700) takes `DATA_LINE_ID` repeated, at most 50, in place of the cursors (a request that carries both ignores the cursors), from a session that negotiated bit 18 as well as bit 4. It is a Get Chat History request like any other, checked under `AccessReadChatHistory`. The reply is the entries of those lines in ascending ID order, under the reader's own read checks: a line the reader cannot read, a line in another channel, and one that does not exist are left out alike, and a tombstone is sent as one. When the entries do not all fit the server's limit on one reply's size, the server drops the highest IDs and sets `DATA_HISTORY_HAS_MORE` to `1`, and the client asks again for the IDs after the last one returned; otherwise `DATA_HISTORY_HAS_MORE` is `0`. The server sends lines, never excerpts embedded in a reply. A client SHOULD fetch quotes only for replies on screen, batched, and a server MAY cache by-ID results.

---

## Notifications

A message's author may ask to be told when someone answers it, so a client can badge a channel the user is not looking at. The [Reactions](Capabilities-Reactions.md#notifications) draft adds a type for reactions. Each session says which kinds it wants.

| Type bit | Name | Told when | To sessions with |
|---|---|---|---|
| 0 | `Replies` | Someone else sends a line whose `DATA_REPLY_TO` names one of the author's lines | Bit 18 |
| 1 | `Reactions` | Defined by the [Reactions](Capabilities-Reactions.md#notifications) draft | Bit 19 |
| 2–31 | | Reserved, for mentions and later kinds | |

- Notifications cover line and ephemeral line targets, whose authors the server knows. For a kept line, the author is its account. For an ephemeral line, the author is the session that sent it, which the server remembers for each line in the [window](Capabilities-Structured-Chat.md#ephemeral-memory).
- A line sent by a guest or another shared account has no one author, and makes no notification.
- A private or instant message's replies already reach its author, the other party, so they make none.
- **Order.** A server MUST NOT notify anyone about a line, or relay anything a companion defines about it, before it has relayed the line itself ([Performance Considerations](#performance-considerations) gives the cheap rule).
- A server never tells an author of their own acts, never tells one about a target they can no longer read, never about a reply they cannot read, and never about a reply naming an ephemeral line its sender was not sent, as for a member who joined after it ([Ephemeral Line IDs](Capabilities-Structured-Chat.md#ephemeral-line-ids)).

### Set Notifications (717)

**Client → Server (request/reply).** Requires bit 18 or 19.

**Request fields:** `DATA_NOTIFY_TYPES` (UInt32), the types this session wants. **Reply fields:** `DATA_NOTIFY_TYPES`, the types the server will send it, leaving out any it does not offer or the session's bits do not allow.

A client MAY also send `DATA_NOTIFY_TYPES` in Login (107), and the login reply then carries the types granted. The choice is the session's and lasts as long as the session. A session that never says holds `Notifications.DefaultTypes`, by default replies alone: a reply is someone speaking to you, and a reaction is not.

### Notification (718)

**Server → Client (notification).** Sent to each live session of the author that asked for the type, or holds it by default.

| Field | Type | Present | Description |
|---|---|---|---|
| `DATA_NOTIFY_TYPE` | UInt16 | Always | The type's bit number |
| `DATA_TARGET` | Binary | Always | The author's line |
| `DATA_NOTIFY_SOURCE` | Binary | Replies, one at a time | The reply, as a [target](#packed-target) |
| `DATA_NOTIFY_COUNT` | UInt32 | Coalesced | How many acts this notification stands for |
| `FieldUserID` (103), `FieldUserName` (102) | | When the actor is on the author's user list | Who acted |

A type may define more fields; [Reactions](Capabilities-Reactions.md#notifications) adds the key.

**Coalescing.** A server SHOULD bound the notifications one author is sent per minute. Past the bound, notifications coalesce: one 718 per target per type when that minute ends, with `DATA_NOTIFY_COUNT` and no source or actor. A server MAY drop notifications beyond a hard cap altogether; the author's client learns of what it missed from history.

An author with no live session is told nothing here. A server MAY hand the same events to its own offline push service, per device, by the types its client registered there, which is outside this draft.

---

## Text Encoding

Bit 18 requires `CAPABILITY_TEXT_ENCODING` (bit 1), so a capable session is a UTF-8 session. Packed targets are **Binary** fields whose text parts are UTF-8 by definition, never transcoded.

---

## Across Server Links

- Link Chat (904) carries the text. A line, ephemeral line, instant-message or private-message reference is not relayed, since those identifiers are one server's. A signed reply's reference, inside its signed object, crosses with the object wherever signed lines and messages do.
- Private messages to or from a linked user carry no `DATA_PM_ID`, and cannot be targets.
- A linked author is never notified.

---

## Data Objects

Within the family's block ([Structured Chat](Capabilities-Structured-Chat.md#data-objects)), beside the chat history block (`0x0F01`–`0x0F1F`). This draft defines these; the family's other drafts define the rest of the block. Integers are big-endian.

| ID (hex) | Decimal | Name | Type | Description |
|---|---|---|---|---|
| `0x0F20` | 3872 | `DATA_LINE_ID` | UInt64 | A line's history ID, repeated in a Get Chat History request by ID ([Lines by ID](#chat-history)) |
| `0x0F21` | 3873 | `DATA_REPLY_TO` | Binary (at most 128) | The [target](#packed-target) a line or message answers |
| `0x0F22` | 3874 | `DATA_REPLY_TARGETS` | UInt16 | Login reply: target kinds kept in replies, `1 << kind` |
| `0x0F23` | 3875 | `DATA_PM_ID` | Binary (17) | A private message's [reference](#private-message-references): `u8 flags` (bit 0 stored), 16 bytes |
| `0x0F24` | 3876 | `DATA_TARGET` | Binary (at most 128) | A target named by a notification, or by a companion's request |
| `0x0F2A` | 3882 | `DATA_CHAT_REASON` | UInt16 | The family's [reason code](#reason-codes) |
| `0x0F2E` | 3886 | `DATA_NOTIFY_TYPES` | UInt32 | [Notification](#notifications) types, a bitmask |
| `0x0F2F` | 3887 | `DATA_NOTIFY_TYPE` | UInt16 | One notification's type |
| `0x0F30` | 3888 | `DATA_NOTIFY_SOURCE` | Binary (at most 128) | The message that caused a notification, as a target |
| `0x0F32` | 3890 | `DATA_NOTIFY_COUNT` | UInt32 | How many acts a coalesced notification stands for |

---

## Reason Codes

`DATA_CHAT_REASON` carries one enumeration for this draft and its companions, which define the codes they add.

| Code | Name | Meaning |
|---|---|---|
| 0 | `OK` | Success |
| 1 | `NoSuchTarget` | The target does not exist, was removed, or is out of the caller's reach; these are not told apart |
| 2 | `NotAllowed` | The caller may not do this |
| 3–5 | | [Reactions](Capabilities-Reactions.md#reason-codes) |
| 6 | `RateLimited` | Slow down |
| 7–11 | | [Server Emoji](Capabilities-Server-Emoji.md#reason-codes) |
| 12 | `InvalidTarget` | A malformed target, or a kind the server does not accept |
| 13 | `TooManyTargets` | More targets than one request may carry |

---

## Performance Considerations

- **Encode once.** A line or message is encoded once for each set of recipients whose bits select the same fields and sub-fields ([Structured Chat](Capabilities-Structured-Chat.md#performance-considerations)), not once per recipient.
- **Decoration.** The target is read by its ID once, before the line is formatted, on the relay path, and only when some reader lacks bit 18; it is not the notification lookup, which comes after commit. A decorated line is then formatted once per class of reader: one more format for `nick`, and for `quote` at most one per access the excerpt depends on.
- **Batch reads.** Quotes for a page's replies are one 700 [by ID](#chat-history), for replies on screen only.
- **Author lookup.** A notification needs one line's author, by the line's own ID. The lookup happens after the line or the companion's act is committed, off the commit path.
- **Ordering.** The cheap rule is a per-channel watermark, the highest line ID whose relay has finished. A notification naming a line above it waits, and a companion's act naming one waits or is refused as for a missing target.
- **Memory.** What this draft remembers of ephemeral lines, their senders, is windowed per chat under [Structured Chat](Capabilities-Structured-Chat.md#ephemeral-memory)'s server-wide cap. Private message references need no memory at all.
- **Notifications** coalesce past the bound and are rate-bound per author.

---

## Security Considerations

- **A reference is not a probe.** The server never tells a sender whether a target exists: scope checks are structural, a reply's private message reference is never checked against the store except to decorate the recipient's own copy, and a by-ID fetch leaves out a line the reader cannot read exactly as a missing one. A sender cannot learn from a reply or a reaction whether its message was read or deleted.
- **References leak nothing about volume.** Ephemeral line IDs count from a random start per chat ([Structured Chat](Capabilities-Structured-Chat.md#security-considerations)), and private message references are random whether or not the message is stored.
- **What the server sees.** A reply's reference is cleartext to the server, beside any message: one outside a sealed box tells the server which message was answered, though not what it said.
- **Decoration copies.** A `quote` excerpt is shown only to a reader who may read the target, and outlives a redaction of it on classic screens; `nick` copies only a name the reader was already shown.
- **Bounded input.** Targets are at most 128 bytes and re-encoded from what was parsed; a by-ID fetch names at most 50 lines.
- **Notifications** reach only an author who asked for them, or holds them by default, and coalesce.

---

## Hotline-ng Binding

*Implementers of the classic extension can skip to [Additions to the Tables](#additions-to-the-tables).*

For a server that also speaks the [Hotline-ng](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng.md) JSON protocol, the same feature as capability `replies`, under the same rules. One store serves both wires.

**Targets** are JSON objects, one key naming the kind:

```jsonc
{ "line": 98124, "room": 2754100983 }            // room absent or 0: the lobby
{ "im": { "login": "alice", "guid": "7d4fa8e2…" } }
{ "pm": "Pz8OKntB1uiCXy1e0G1x4A" }                // a stored private message, by its reference
{ "signed": { "signer": "6htgz65…", "id": "…", "hash": "…" } }
```

`pm` is the 16-byte reference of the classic `DATA_PM_ID`, spelled base64url without padding, for the same message. A `msg` event and the `msg` reply carry it as `ref`. On this wire only a stored message has one for now; a message delivered live and never stored has no `ref`, and cannot be named, until the ephemeral reference arrives here too. A stored message's `id` stays what it is today, a cursor for `inbox` paging and `msg_read`, and is never a target. Hotline-ng has no private chats yet, so it has no counterpart to kind 4; when it gains them, their lines carry the chat's ephemeral ID under the same rules.

**Replies.** `chat` and `msg` take `re`, a target. The `chat`, `msg` and `im` events, `history` and `inbox` rows, and the messages of a `requests` listing carry it as kept. `re` naming a line in another room, or an `im` naming neither the sender nor the recipient, is refused `bad_request`, so the client learns its reply did not take; the classic wire, which has no refusal, drops it instead. The checks are the same structural ones, and a `pm` is always accepted. A signed line or message carries `re` inside its signed object. The login block `replies`: `{ "targets": ["line", "im", "pm"] }`. `history` takes `ids` (1–50 line ids) in place of the cursors, to fetch a page's quoted lines in one request, and drops the highest ids with `has_more` when they do not fit.

Decoration is the classic wire's alone: an ng event's `text` is always as sent.

**Notifications.** With `replies` or `reactions`, `login` takes `notify: ["replies", "reactions"]`, and so does a `notify` request (`{ "types": [...] }`, a read), answered `{ "notify": [ … ] }` with the types granted, or `bad_request` for a type it does not know. Event `notification`: `{ "type": "reply" | "reaction", "target", "source"?, "key"?, "count"?, "from"?: { "uid", "nick" } }`, `count` for a coalesced one. A server's push registrations may name the same types; that is the push service's to define.

---

## Additions to the Tables

All **proposed, pending allocation**. None is used for anything else in any published document at the time of writing.

Capability bit, as a row of [Capabilities.md](Capabilities.md#defined-capability-bits):

| Bit | Mask | Name | Description |
|---|---|---|---|
| 18 | `0x00040000` | `CAPABILITY_REPLIES` | **Draft**, not adopted. A line or message may name the one it answers in `DATA_REPLY_TO`, a chat line by the ID its [structured line](Capabilities-Structured-Chat.md) carries, and private messages carry references of their own. Only alongside bits 1 and 17, and never on a server link (bit 11). |

Transactions, from the block 710–719 the [Structured Chat](Capabilities-Structured-Chat.md#additions-to-the-tables) draft proposes be kept for this family:

| ID | Name | Type | Description |
|---|---|---|---|
| 717 (`0x02CD`) | Set Notifications | Request/reply | Choose the notification types a session is sent |
| 718 (`0x02CE`) | Notification | Server → client | A reply to, or a companion's act on, the author's line |

Fields, from the block `0x0F20`–`0x0F4F` kept for this family: `0x0F20`–`0x0F24`, `0x0F2A`, `0x0F2E`–`0x0F30` and `0x0F32`, as in [Data Objects](#data-objects).

Chat history: sub-field `0x0020` `REPLY_TO`, in the range `0x0020`–`0x002F` kept for this family; and repeated `DATA_LINE_ID` in a Get Chat History request ([Lines by ID](#chat-history)), a change to that extension's request.

Reason codes 0–2, 6, 12 and 13, in the family's enumeration; notification type 0.

---

## Open Questions

1. **Where the numbers live.** Would you rather they sat inside the chat history extension's space: replies as its proposed `Reply-To ID` sub-field `0x0002`, the transactions in 705–709? 705–709 is five numbers, and a report transaction has been asked for there too, so this family proposes a block of its own. Where the line itself lives is asked in the [Structured Chat](Capabilities-Structured-Chat.md#open-questions) draft.
2. **Private message references.** No extension tells a classic client that it is reading a stored private message. Should `DATA_PM_ID` be this draft's, or belong to an offline-message extension of its own?
3. **Sealed messages.** Should a sealed message's reply reference, and the [Server Emoji](Capabilities-Server-Emoji.md) draft's refs, travel inside the sealed body, hidden from the server and checked by the recipient? That is the messaging identity draft's to agree.
