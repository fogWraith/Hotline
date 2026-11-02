# Draft: Structured Chat

> **Status:** Draft proposal by Misha Nasledov (hxd-ng). **Not adopted**: nothing here is part of the protocol unless another document says so. It proposes capability bit 17; field ID `0x0F33`, with a row for it in the Text Encoding Extension's embedded binary fields table; on Chat Message (106), `FieldUserID` (103), which servers descended from hxd already send, and `DATA_CHANNEL_ID` (`0x0F01`); and in Chat History, entry flag bit 3 (set only on a live line) and three rules for every entry: the nick capped as the user list caps it, the text after line-ending normalization, and a message cut at a character boundary to fit. It also proposes that transactions 710–719, field IDs `0x0F20`–`0x0F4F` and history sub-field types `0x0020`–`0x002F` be kept for this draft and the three that go with it, the [Replies](Capabilities-Replies.md), [Reactions](Capabilities-Reactions.md) and [Server Emoji](Capabilities-Server-Emoji.md) drafts. All of these were unassigned at the time of writing; none is allocated. Until they are, they are used only between servers and clients whose operators and users have chosen to try it, and nothing here changes what is asked of a server or client that has not.

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

On the classic wire, a live chat line reaches a client only as the text the server formatted for it: Chat Message (106) carries `FieldData` (101) as `\r%13.13s:  %s`, or `\r *** nick action` for an emote, and on servers descended from hxd the sender's user ID, but nothing else about the line. A client that wants the sender's nick, the text or the time has to parse them back out of that string, which fails for a nick longer than the 13-column field, a nick containing `:`, and two users with the same nick. It gets no line ID and no timestamp at all, though a [Chat History](Capabilities-Chat-History.md) entry for the same line has both.

This extension sends the line as data as well. To a session that asks, every 106 that relays a line also carries the Chat History entry for that line, live: its ID, the time the server received it, its flags, the sender's icon and nick, and the text as sent. `FieldData` is unchanged by this draft, so a client that ignores the new field loses nothing, and a client that does not ask sees no change at all.

It is the foundation of two drafts, its companions, that each attach something to a line: [Replies](Capabilities-Replies.md) names the line a message answers, and [Reactions](Capabilities-Reactions.md) puts emoji on a line. They name lines by the IDs this draft gives them, and Replies carries what it adds on 106 as a sub-field of the entry. A sibling, [Server Emoji](Capabilities-Server-Emoji.md), marks a server's emoji in text; it stands alone, and uses the entry's raw text where a client has it. For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md).

## Table of Contents

- [What Does Not Change](#what-does-not-change)
- [Negotiation](#negotiation)
  - [Capability Bit](#capability-bit)
  - [Server Configuration](#server-configuration)
  - [Login Reply](#login-reply)
- [The Structured Line](#the-structured-line)
  - [Beside the Entry](#beside-the-entry)
  - [Why One Packed Field](#why-one-packed-field)
  - [Example](#example)
- [Line IDs](#line-ids)
  - [Kept Lines](#kept-lines)
  - [Ephemeral Line IDs](#ephemeral-line-ids)
  - [Lines Sent to One Session](#lines-sent-to-one-session)
  - [Line IDs and Volume](#line-ids-and-volume)
  - [Ephemeral Memory](#ephemeral-memory)
- [Which Lines](#which-lines)
- [Sending (105)](#sending-105)
- [The Sender's Echo](#the-senders-echo)
- [Chat History](#chat-history)
- [Text Encoding](#text-encoding)
- [Other Extensions](#other-extensions)
  - [Colored Nicknames](#colored-nicknames)
  - [Inline Media](#inline-media)
  - [Server Links](#server-links)
  - [Signed Lines](#signed-lines)
- [Data Objects](#data-objects)
- [Performance Considerations](#performance-considerations)
- [Security Considerations](#security-considerations)
- [Hotline-ng Binding](#hotline-ng-binding)
- [Additions to the Tables](#additions-to-the-tables)
- [Open Questions](#open-questions)

---

## What Does Not Change

- **`FieldData` on 106 is unchanged by this draft (the [Replies](Capabilities-Replies.md#how-other-readers-see-a-reply) draft lets a server decorate it for sessions without bit 18).** A capable session is sent the same formatted text as any other session of its class, byte for byte, beside the new field. A client that draws the formatted text and ignores the rest works as it does today.
- **A client that does not negotiate bit 17 is sent no new field.** No field is added to anything it is sent. Two rules here apply to all of a server's sessions, and both only shrink or tidy values it already sends: the three [entry rules](#chat-history) for every 700 entry, and `FieldUserID` 0 for a speaker hidden from the recipient's user list ([Security Considerations](#security-considerations)).
- **Send Chat (105) is unchanged.** Its `FieldData` was always the raw text; this draft only hands it back.
- **The structured line is never parsed back out of the formatted text.** It is built from what the server already knows when it formats the line.
- **Private chats are never stored.** Their lines are numbered in memory, and the Chat History Extension's rule that private chats are not persisted holds unchanged.
- **No new transaction.** Lines arrive on 106 and history on Get Chat History (700), as today.

---

## Negotiation

### Capability Bit

| Bit | Mask | Name | Description |
|---|---|---|---|
| 17 | `0x00020000` | `CAPABILITY_STRUCTURED_CHAT` | Every 106 that relays a line to the session also carries the line as a Chat History entry in `DATA_CHAT_LINE`, with the sender's `FieldUserID` |

This is a bit above 15, so a client that sets it sends `DATA_CAPABILITIES` in four bytes, and a server that confirms it answers in four ([Field Width](Capabilities.md#field-width)).

- A server MUST NOT confirm bit 17 on a server link (bit 11). Links carry raw text already, in Link Chat (904).
- A server MAY confirm bit 17 to a messenger session (bit 8). It is harmless: such a session is in no chat, and receives nothing to use it on.
- Bit 17 does not require bit 1 (`CAPABILITY_TEXT_ENCODING`) or bit 4 (`CAPABILITY_CHAT_HISTORY`). Its entry is written in the session's encoding, as a 700 entry is ([Text Encoding](#text-encoding)), and a client can use live structure without fetching history; such a client may be sent Chat History's legacy replay ([Which Lines](#which-lines)). A bit 17 session without bit 4 thus receives an entry in `DATA_CHAT_LINE`, a touch on Chat History's rule that a client without bit 4 never receives history-related fields. The drafts built on it require bit 1 for their own reasons.
- The drafts built on it require bit 17: a server MUST NOT confirm their bits (18 and 19) unless it also confirms bit 17. Server Emoji's bit 20 does not.

### Server Configuration

Recommended settings (implementation-defined):

| Setting | Type | Default | Description |
|---|---|---|---|
| `StructuredChat.Enabled` | bool | `false` | Confirm bit 17 to clients that ask. It costs a server a counter per chat and the larger frames ([Performance Considerations](#performance-considerations)) |
| `StructuredChat.EphemeralWindow` | int | `200` | Used only with bits 18 or 19. How many of a chat's most recent ephemeral lines those drafts remember anything about |
| `StructuredChat.EphemeralMaxBytes` | int | `16777216` | Used only with bits 18 or 19. Server-wide cap on everything those drafts hold in memory for ephemeral lines ([Ephemeral Memory](#ephemeral-memory)) |

This draft holds nothing per line itself, only a counter per chat. The window and the cap bound what the drafts built on it keep about ephemeral lines, and mean nothing on a server that offers neither; they are defined here once so that both drafts are held to the same bounds.

### Login Reply

Nothing beyond the confirmed bit. Every structured line says whether its ID is a history ID, an ephemeral one, or `0` for a line no one else was shown ([Line IDs](#line-ids)), so a client needs no server-wide statement of how lines are numbered.

---

## The Structured Line

To a session that negotiated bit 17, every 106 that relays a line MUST carry `DATA_CHAT_LINE`: one [Chat History entry](Capabilities-Chat-History.md#history-entry-format), in exactly the layout of `DATA_HISTORY_ENTRY`, describing that line.

| Entry field | On a live line |
|---|---|
| Message ID | The line's [ID](#line-ids): its history ID if the server kept it, otherwise an ephemeral ID |
| Timestamp | When the server received the line, or for a line the server writes, when it wrote it, as Unix seconds (int64, UTC): the same value its history entry has, where it was kept. A line the server writes for each recipient may differ by a second between them |
| Flags | Chat History's flags, with their meanings: `is_action` (bit 0) for an emote, `is_server_msg` (bit 1) for a line the server wrote, and `is_ephemeral` (bit 3, proposed here) when the ID is an ephemeral one. `is_action` is never set on a server line, though a notice drawn `\r *** text` looks like an emote without a nick, and `is_deleted` (bit 2) is never set on a live line |
| Icon ID | The sender's icon when the line was relayed; `0` for a server line |
| Nick | The sender's nick as this recipient's user list shows it, unpadded and uncut by the formatter: what the client would otherwise parse out of `FieldData`. It is capped as this server's user list caps a nick (31 bytes on mhxd-descended servers), after conversion and at a character boundary, a ghost's tag kept by cutting the name before it, here and in the line's 700 entry alike ([Chat History](#chat-history)). For a server line, the same as its 700 entry: empty, or the server name where Chat History records one |
| Message | The text the server relays, unformatted: for an emote, the action without the nick. It is the text the sender sent in 105, unless the server's own filters changed it, in which case it is the changed text that `FieldData` shows too. It never carries the decoration the [Replies](Capabilities-Replies.md#how-other-readers-see-a-reply) draft lets a server add to `FieldData` for a session without bit 18 |
| Sub-fields | Every sub-field a 700 entry for this line would carry to this session ([Chat History](#chat-history)), except `EMOJI_REFS`, which 106 carries as `DATA_EMOJI_REFS` |

A server builds the entry with the same encoder it uses for 700, so that a live line and its history entry cannot disagree. One 105 is one line: a server that formats a multi-line send into one 106 sends one entry, whose message keeps its line breaks, in the session's line ending as the Text Encoding Extension's [line-ending normalization](Capabilities-Text-Encoding.md#line-ending-normalization) gives it, on 106 and in 700 alike.

### Beside the Entry

The entry says what a line is, not where it was said or by which session, since a history entry has neither. Those go beside it, as fields of the 106 itself:

| Field | Present | Description |
|---|---|---|
| `FieldUserID` (103) | Always, to a capable session | The sender's user ID on this server, `0` for a server line. Servers descended from hxd already send it on every 106, to every session; this draft makes it required for a capable session and gives it a meaning for every line |
| `DATA_CHATID` (114) | For a line in a private chat, as today | The chat the line was said in |
| `DATA_CHANNEL_ID` (`0x0F01`) | For a line in a persistent channel, as below | The channel whose history holds the line |

A user ID is a fact about the live session, recycled after the user leaves, so it never belongs in a history entry; on 106 it is what a client needs to find the sender in its user list, which a nick alone cannot do.

**Channels.** Chat History's [note on channel 0](Capabilities-Chat-History.md#channel-0-public-chat) foresees `DATA_CHANNEL_ID` on 106 once named channels exist. This draft fixes how: a capable session's 106 for a line in a persistent channel carries `DATA_CHANNEL_ID` when the session addresses the channel as a channel, under whatever extension defines that, and never for channel 0: a 106 with neither field is public chat. A server that relays a persistent channel to classic clients as a private chat sends its lines under `DATA_CHATID`, and MAY add `DATA_CHANNEL_ID` beside it, only where 700 answers that channel, so that a capable client can fetch that chat's history. A kept line there carries its history ID with `is_ephemeral` clear, which is how a client tells a channel's line from an ephemeral private chat's; a line the server does not keep there is numbered ephemerally, as in any chat.

### Why One Packed Field

The alternative is a field per fact: the ID, the time, the flags, the nick, the icon and the raw text each on 106 in a field of its own. One packed entry is recommended instead:

- **One parser and one encoder.** A client that reads Chat History already parses this layout, and a server already writes it. Nothing about a line is defined twice.
- **Live and history cannot disagree.** The 106 entry and the 700 entry for a line come from one encoder and one record, so a client deduplicates a catch-up page against what it saw live by ID, and finds the same nick, time and text under it.
- **Extension in one place.** A draft that attaches something to a line defines one history sub-field, and it travels on 106 and in 700 alike. With separate fields, every addition would need a 106 field and a sub-field, kept in step.
- **Size.** An entry costs 28 bytes beyond its nick and text, field header included; the same facts in separate fields cost about 16 bytes more in headers. Neither matters beside the text ([Performance Considerations](#performance-considerations)).

What separate fields would have that the entry lacks is per-field transcoding by a server that converts fields from a table ([Text Encoding](#text-encoding)), and a server that builds the entry per encoding, as it does for 700, does not need it.

### Example

alice (user ID 7, icon 128) says `hi` in public chat, kept as history ID 1000 at 2026-11-01T12:00:00Z:

```
 HEADER (20 bytes)    Type = 106 (0x006A)

 PAYLOAD (66 bytes)
 00 03                         ParamCount = 3

 00 65  00 13                  FieldData (101), 19 bytes
 0D 20 20 20 20 20 20 20 20    "\r        alice:  hi", as formatted today
 61 6C 69 63 65 3A 20 20 68 69

 00 67  00 02  00 07           FieldUserID (103) = 7

 0F 33  00 1F                  DATA_CHAT_LINE, 31 bytes
 00 00 00 00 00 00 03 E8       message_id = 1000
 00 00 00 00 6A E7 29 C0       timestamp  = 1793534400 (2026-11-01T12:00:00Z)
 00 00                         flags      = 0
 00 80                         icon_id    = 128
 00 05  61 6C 69 63 65         nick       = "alice"
 00 02  68 69                  message    = "hi"
```

The whole transaction is 86 bytes; without the entry it is 51.

---

## Line IDs

Every structured line a chat's readers share has an ID, so that the drafts built on this one can name any line a client was shown. A line sent to one session alone has ID `0` ([Lines Sent to One Session](#lines-sent-to-one-session)).

### Kept Lines

A line the server keeps in chat history carries its history ID, the one Get Chat History gives it, with `is_ephemeral` clear. It is opaque and, as Chat History says, compared for order and equality only. Today a classic client learns a line's ID only from 700; with bit 17 it learns it as the line arrives.

### Ephemeral Line IDs

A line the server does not keep carries an ephemeral ID, with `is_ephemeral` set: every line of a private chat, and every public chat line on a server without history or that it otherwise does not keep, such as a notice it sends to the whole chat.

- The ID comes from a counter of that chat's own, public chat being chat 0. The counter starts at a value drawn uniformly at random below 2^62, no later than the chat's first line, so it cannot wrap, and goes up by one per ephemeral line relayed. It is never stored, and it ends when the chat does or the server restarts.
- The ID means something only together with its chat ID. A chat ID reused later starts a new counter at a new random value, and the window the drafts built on this one apply refuses a stale ID anyway.
- A server MUST NOT number ephemeral lines from a counter shared across chats, which would show every member how busy the server is. A per-chat counter with a random start shows nothing beyond the chat's own lines ([Line IDs and Volume](#line-ids-and-volume)).
- A member is given no ID for a line relayed before they joined, since they were never sent it. The drafts built on this one treat such a line as one the member does not hold, though its ID can be guessed from the ones around it: a reaction to it is `NoSuchTarget`, and a reply naming it notifies no one.
- An ephemeral ID never appears in Get Chat History. On a server that keeps history, chat 0's ephemeral IDs and its history IDs are separate spaces, told apart by the flag, and never compared with each other.

### Lines Sent to One Session

A line the server sends to one session alone, such as a command's output or a notice to the user it concerns, carries ID `0`, with `is_ephemeral` clear, meaning that it cannot be named. It takes no number from its chat's counter: a reaction to a line nobody else was shown would otherwise publish, to every reader of the chat, a target they never saw. The drafts built on this one answer a target with line ID `0` as `NoSuchTarget`, or drop it where they have no answer to give.

### Line IDs and Volume

A line ID SHOULD NOT reveal how busy a conversation is that its reader cannot see. Ephemeral IDs meet this by construction. A history ID meets it when the server numbers each channel's history from a counter of its own, or by any scheme without the leak. A server whose history IDs come from one counter across channels lets a reader of one channel count the lines of the others from the gaps; that is a leak the server accepts when it chooses such a counter. This stays a SHOULD because a MUST would cost such a server a migration: per-channel ID spaces mean re-keying everything that names a line by its ID, redactions, reports, purges and reactions among them.

### Ephemeral Memory

The drafts built on this one may remember something about recent ephemeral lines: who sent each and the nick it was drawn under, for [Replies](Capabilities-Replies.md#notifications)' notifications and its [decoration](Capabilities-Replies.md#how-other-readers-see-a-reply), or its reactions, for [Reactions](Capabilities-Reactions.md#reaction-changed-711). They keep it for a chat's most recent `StructuredChat.EphemeralWindow` lines, and everything they hold, together, is held to `StructuredChat.EphemeralMaxBytes` across the server. Each chat's window is charged as a share of it. Past the cap, the server trims first the holders whose share is above the average, oldest lines first.

---

## Which Lines

A structured line goes with every 106 a capable session is sent for a line of chat, and with no other.

| Line | ID | `FieldUserID` | Flags | Beside it |
|---|---|---|---|---|
| A user's public chat line | History ID if kept, otherwise ephemeral, chat 0 | The sender's | `is_ephemeral` if its ID is | Nothing: public chat |
| A user's line in a private chat | Ephemeral | The sender's | `is_ephemeral` | `DATA_CHATID` |
| A user's line in a persistent channel shown as a private chat | History ID if kept, otherwise ephemeral | The sender's | `is_ephemeral` if its ID is | `DATA_CHATID`, and MAY `DATA_CHANNEL_ID` where 700 answers the channel |
| An emote (`/me`) | As for its chat | The sender's | `is_action` | As for its chat |
| A line the server writes into a chat for its readers: a notice, a join or leave line where a server prints one | History ID if kept, otherwise ephemeral | The user it is about, or `0` | `is_server_msg`, and `is_ephemeral` if its ID is | As for its chat |
| A line sent to one session alone: a command's output, a notice to the user it concerns | `0` ([Lines Sent to One Session](#lines-sent-to-one-session)) | As for a server line | `is_server_msg` if it is drawn as a notice | As for its chat |
| A line from a linked server's user ([Server Links](#server-links)) | As for public chat | The ghost's user ID on this server | | Nothing: only public chat crosses a link |
| A line replayed as Chat History's [legacy broadcast](Capabilities-Chat-History.md#legacy-broadcast-behaviour) | None: not a structured line | | | |

- **`is_server_msg` follows how the line is drawn.** It is set on a line the server formats as a notice rather than as `nick:  text`, and `FieldUserID` then MAY name a user behind it, as mhxd names the kicker on a kick notice and the kicked user on a flood kick, or be `0`, as for a command's output. A user run by the server, such as a bot or a system account on the user list, speaks as a user, its own user ID and nick with `is_server_msg` clear, only where its line is drawn as `nick:  text`. A notice drawn as `\r *** text` is a server line even when it comes from a system account's user ID.
- **`FieldUserID` 0 with `is_server_msg` clear** means a speaker who is not on the recipient's user list ([Security Considerations](#security-considerations)).
- **An administrator's broadcast** (355) reaches clients as Server Message (104), not as a chat line, and is not covered. A server that records broadcasts in history, as Chat History allows, gives them history IDs a client sees in 700 alone.
- **A legacy replay carries no entry.** A session with bit 4 SHOULD NOT be replayed to, as Chat History says. One with bit 17 and not bit 4 cannot use 700, and MAY be. The replayed 106s carry no `DATA_CHAT_LINE`, which is how it tells them from live lines.

---

## Sending (105)

Nothing changes in Send Chat (105). Its `FieldData` is the raw text, its `FieldChatOptions` (109) says whether it is an emote, and `DATA_CHATID` names a private chat, as today; the structured line on 106 is built from them.

**Fields a companion adds.** A draft built on this one MAY define a field a capable client adds to 105, such as the [Replies](Capabilities-Replies.md#replies) draft's `DATA_REPLY_TO` or, later, a signature ([Signed Lines](#signed-lines)). Such a draft defines, in one place, both the field on 105 and the history sub-field that carries what the server kept of it, on 106 and in 700 alike. The server checks the field as that draft says, keeps it wherever it keeps the line, and drops it from a sender that did not negotiate that draft's bit, relaying the rest. A field a server does not know is dropped, never passed on: a server relays only what it parsed and kept.

---

## The Sender's Echo

105 has no reply. A server already sends the sender its own line as a 106; to a capable sender that echo carries the structured line, ID included, and it is how a sender learns the ID of what it said. A server MUST send a capable sender its echo for every line it relays, even where it would otherwise skip it: the case this exists for is a sender that may send to a chat but not read it, which today gets no echo.

A client matches echoes to its sends in order, chat by chat: one connection's lines to one chat are relayed in the order sent, and a server that commits each chat separately may relay lines to two chats in either order. A line the server drops, for flooding or because the chat is read-only, has no echo, so a client SHOULD treat a pending send to a chat whose echo has not come before the echo of a later one to the same chat as dropped.

---

## Chat History

A 700 entry and a live structured line for the same kept line describe it alike: the same ID, timestamp, flags, icon, nick and text, from one encoder. For that to hold, a server that confirms bit 17 applies three rules to every 700 entry it sends, to every session, as it does to the live entry: the nick is capped as the user list caps it, the text follows line-ending normalization, and a message too long for its entry is cut at a character boundary. Other Chat History servers MAY apply them; these are touches on Chat History ([Open Questions](#open-questions)). A client that has a line from 106 and receives it again in a catch-up page keeps one copy, matched by the conversation and the ID, `is_ephemeral` clear. A line with `is_ephemeral` set never appears in 700 and is never matched against it.

**Sub-fields.** The entry's [optional sub-fields](Capabilities-Chat-History.md#optional-sub-fields-mini-tlv) are one registry for 106 and 700:

| Range or type | Defined by | Sent to sessions with |
|---|---|---|
| `0x0001`–`0x000F` | Chat History, core | As that extension defines |
| `0x0010`–`0x001F` (`0x0010`–`0x0014` allocated) | [Inline Media](Capabilities-Inline-Media.md), the attachment | Bit 3, as Chat History says |
| `0x0020` `REPLY_TO` | [Replies](Capabilities-Replies.md#chat-history) | Bit 18 |
| `0x0021` `REACTIONS`, `0x0023` `REACTIONS_MINE` | [Reactions](Capabilities-Reactions.md#chat-history), in 700 alone: a live line has none yet | Bit 19 |
| `0x0022` `EMOJI_REFS` | [Server Emoji](Capabilities-Server-Emoji.md#server-emoji-in-text), in 700 alone: on 106 its refs travel as `DATA_EMOJI_REFS` beside the text | Bit 20 |
| `0x0024`–`0x002F` | Kept for this family | |

- A sub-field goes to a session that negotiated the bit of the document that defines it, on 106 and in 700 alike, and a reader skips one it does not know, as Chat History's parsing rule says.
- A tombstoned 700 entry carries none of the family's sub-fields.
- They count toward the server's limit on one 700 reply's size, the transaction it must fit in, as the entries do. A structured line on 106 is one entry in one field, at most 65535 bytes. A chat line's own bounds keep it well under that, but conversion can lengthen text, Mac Roman to UTF-8 up to three times, so a server cuts the message at a character boundary where the entry would not fit, on 106 and in 700 alike.

**The flag.** `is_ephemeral` is flag bit 3 of the entry, which Chat History reserves as zero. This draft proposes it for a live line alone; a server never sets it in a 700 entry, and a history reader that ignores unknown flags, as Chat History requires, is unaffected.

---

## Text Encoding

The entry's nick and message are in the session's negotiated encoding, UTF-8 with bit 1 or Mac Roman without it, exactly as in a 700 entry, with `?` for a character Mac Roman cannot write. The text has the line endings the session is sent elsewhere ([Line Ending Normalization](Capabilities-Text-Encoding.md#line-ending-normalization)). Every cap applies after conversion, at a character boundary, as it does for the formatted line.

`DATA_CHAT_LINE` is an embedded binary field: a server builds it for each encoding it sends, as it builds the formatted line, and never transcodes it whole ([Embedded Binary Fields](Capabilities-Text-Encoding.md#embedded-binary-fields)).

**Why not require bit 1.** The problem this draft solves, a sender and a text a client cannot recover, is the same in Mac Roman, and Chat History already serves Mac Roman sessions this layout. A client written for a classic Mac gets the sender's user ID and an exact nick without speaking UTF-8. What does need UTF-8 needs it for its own reasons and says so: the drafts built on this one carry UTF-8 binary fields, and a signature covers exact UTF-8 text ([Signed Lines](#signed-lines)).

---

## Other Extensions

### Colored Nicknames

Unchanged. A structured line carries no color: a client colors the sender by the `DATA_COLOR` its user list holds for that user ID, which the formatted text never let it find reliably. A color recorded with a line is Chat History's proposed core sub-field `0x0001`, and that extension's to allocate.

### Inline Media

Unchanged. A line with an attachment carries Inline Media's fields on 106 to a bit 3 session, as that extension says, and its entry carries the attachment's sub-fields `0x0010`–`0x0014` to the same session, as a 700 entry does. Both come from the one record of the line, so they agree; a client with bits 3 and 17 MAY read either. A recipient without bit 3 gets the entry with `[image]` for an otherwise empty message, as in history, and is never sent the handle.

### Server Links

- A server link does not negotiate bit 17. Link Chat (904) already carries a line's raw text, and the receiving server formats it for each client ([Link Chat](Capabilities-Server-Link.md#link-chat-904)), so it builds a ghost's structured line the same way it builds a local user's.
- A ghost's line carries the ghost's user ID on this server, the one its user list entry has, and the ghost's display name as this server shows it, with the home server's tag where the operator turned the tag option on. The text is 904's `FieldData`, converted for the session.
- Its ID is this server's: the history ID it kept the line under, or an ephemeral one.
- Chat History keeps a ghost's line under the display name it had when it was logged ([Ghosts and Other Extensions](Capabilities-Server-Link.md#ghosts-and-other-extensions)), so the live entry and the 700 entry agree when the line is relayed. If the operator later changes the tag option, history keeps the name the line was logged under, and a client that holds both trusts the 700 entry for that line. `DATA_LINK_LINE_ID`, which names a line across the network, is not carried in this version ([Open Questions](#open-questions)).
- Only public chat crosses a link, so a ghost's line never carries `DATA_CHATID` or `DATA_CHANNEL_ID`.

### Signed Lines

The [user keys draft](Server-Link-User-Keys.md) notes that nothing yet delivers a signed line to a classic client: a ghost's chat reaches it as 106, which carries no signature. A signature covers the text the sender typed, and the formatted 106 cannot be turned back into it. The structured line is what makes a binding possible: it carries that text on 106 and in 700, so a later draft can add the signature as a field a client adds to 105 and a sub-field of the entry, and a reader can check one against the other. This draft defines no signing. A binding that does will require bit 1, since a signature covers exact UTF-8.

---

## Data Objects

The family's field IDs are in `0x0F20`–`0x0F4F`, beside the chat history block (`0x0F01`–`0x0F1F`). This draft defines one; the drafts built on it define the rest of the block. Integers are big-endian.

| ID (hex) | Decimal | Name | Type | Description |
|---|---|---|---|---|
| `0x0F33` | 3891 | `DATA_CHAT_LINE` | Binary (at most 65535) | One live line, as a [Chat History entry](Capabilities-Chat-History.md#history-entry-format), on Chat Message (106) to a session with bit 17 |

Existing fields given a role on 106: `FieldUserID` (103), the sender; `DATA_CHATID` (114), as today; `DATA_CHANNEL_ID` (`0x0F01`), a persistent channel ([Beside the Entry](#beside-the-entry)).

Chat History entry flags:

| Bit | Mask | Name | Description |
|---|---|---|---|
| 3 | `0x0008` | `is_ephemeral` | The message ID is an [ephemeral ID](#ephemeral-line-ids), not a history ID. Set only on a live line, never in a 700 entry |

---

## Performance Considerations

- **Size.** A structured line adds 28 bytes plus the nick and the text to a 106, and 6 more for `FieldUserID` where a server does not already send it. The formatted text is about the text plus 17 bytes, so a capable 106 is roughly twice the size of a classic one: 86 bytes against 51 for a two-letter line, and about 285 against 150 for a line of a hundred bytes from an eight-letter nick. That is acceptable because chat is small. A lobby busy enough to scroll several lines a second costs a capable reader well under a kilobyte a second more, a fraction of any file transfer, and in exchange no client has to guess who spoke. A server that bounds each connection's queue in bytes counts the larger frames, so a slow reader is dropped no later than before, by the same bound.
- **Encode once.** A line's entry is the same bytes for every capable session that shares an encoding and the bits that select its sub-fields, and most lines carry no sub-field, so a server MAY build one entry per encoding per line. Building it per recipient is acceptable too: it costs the same order as formatting the line, which a server already does.
- **No lookups.** Everything in the entry is what the server holds when it formats the line. The ID is the one it just logged under, or the next value of the chat's counter.
- **Memory.** Ephemeral IDs are a counter per chat. What the drafts built on this one remember about ephemeral lines is windowed per chat, charged as a share of a server-wide cap and trimmed from the largest holders first ([Ephemeral Memory](#ephemeral-memory)).

---

## Security Considerations

- **Nothing new about the sender.** A structured line carries what the recipient's user list and the formatted line already show: a user ID, a nick and an icon. A server MUST NOT put a speaker on a structured line in a way the recipient's user list does not show them: a speaker who is not on it, such as one the server keeps out of the list, is sent with `FieldUserID` 0, `is_server_msg` clear, and the nick the formatted line shows. That `FieldUserID` 0 goes on every 106 to that recipient, capable or not: a period session that was still sent the real user ID would make the rule protect nothing. A user a server line names is one the recipient's list shows, or `0`.
- **User IDs.** They are recycled, so a client MUST NOT treat one as naming a person beyond the session it saw on the user list, and never keeps it with a line it stores. The entry deliberately has none.
- **Timestamps** are the server's receive time, or for its own lines its write time, which Chat History already serves to readers of the same lines.
- **Line IDs.** A session with bit 17 is sent a kept line's history ID whether or not it may use 700, so it learns IDs it could not have fetched; an ID is opaque, and names a line the session was sent anyway. What an ID can reveal is volume, held to [Line IDs and Volume](#line-ids-and-volume). Ephemeral IDs count from a random start per chat, and a line sent to one session takes no number at all.
- **Private chats are never stored.** Their IDs live in a counter, and what the drafts built on this one keep about their lines lives in memory for the life of the chat, never written anywhere.
- **Bounded input.** Nothing here is sent by a client. A field a companion adds to 105 is parsed, bounded and re-encoded by that companion's rules.

---

## Hotline-ng Binding

*Implementers of the classic extension can skip to [Additions to the Tables](#additions-to-the-tables).*

The [Hotline-ng](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng.md) JSON protocol already carries a line as data, so it needs no capability for this: its `chat` event is the structured line.

| Entry, or beside it | `chat` event |
|---|---|
| Message ID | `id`, present when the server kept the line. Hotline-ng has no ephemeral IDs yet; an unkept line has no `id` |
| Timestamp | `at`, integer Unix seconds |
| `is_action` | `style: "action"` |
| `is_server_msg` | A `notice` event in place of a `chat` |
| Icon ID | The sender's icon, from the roster by `from.uid` |
| Nick, `FieldUserID` | `from.nick`, `from.uid` |
| Message | `text` |
| `DATA_CHATID`, `DATA_CHANNEL_ID` | `room`, absent for the lobby |
| The family's sub-fields, and Server Emoji's refs | `re`, `emoji`, and on `history` lines `reactions` and `reactions_mine`, by the bindings of the drafts that define them; Inline Media's as `media` |

One store serves both wires, so a line said on one has the same `id` on the other.

---

## Additions to the Tables

All **proposed, pending allocation**. None is used for anything else in any published document at the time of writing.

Capability bit, as a row of [Capabilities.md](Capabilities.md#defined-capability-bits):

| Bit | Mask | Name | Description |
|---|---|---|---|
| 17 | `0x00020000` | `CAPABILITY_STRUCTURED_CHAT` | **Draft**, not adopted. Every Chat Message (106) that relays a line also carries it as a Chat History entry in `DATA_CHAT_LINE` (`0x0F33`): its ID, time, flags, icon, nick and raw text, with the sender's `FieldUserID`. `FieldData` is unchanged by this draft. Never on a server link (bit 11). |

Field `0x0F33` `DATA_CHAT_LINE`, as in [Data Objects](#data-objects), and as a row of the Text Encoding Extension's [Embedded Binary Fields](Capabilities-Text-Encoding.md#embedded-binary-fields) table: nick and message, each after a `u16` length, at the offsets of a `DATA_HISTORY_ENTRY`. The field MUST be built per encoding, as `DATA_HISTORY_ENTRY` is; a transcoder that converts it in place applies the nick cap, line endings and cut-to-fit after converting. `DATA_HISTORY_ENTRY` (`0x0F05`) has no row in that table either, and this draft offers one for both. Chat History: entry flag bit 3 `is_ephemeral`, a change to that extension's reserved flags; the sub-field range `0x0020`–`0x002F`; and, so that live and history agree, three rules for every entry a server that confirms bit 17 sends, which other Chat History servers MAY apply: the nick capped as the user list caps it, the text after line-ending normalization, and a message cut at a character boundary to fit ([Chat History](#chat-history)).

**The family's blocks.** Transactions 710–719, the free block after chat history's 700–709 and before inline media's 750; fields `0x0F20`–`0x0F4F`, beside chat history's `0x0F01`–`0x0F1F`; and history sub-field types `0x0020`–`0x002F`, beside core (`0x0001`–`0x000F`) and inline media (`0x0010`–`0x001F`). Kept for this draft, the two built on it and Server Emoji:

| Draft | Bit | Transactions | Fields | Sub-fields |
|---|---|---|---|---|
| Structured Chat | 17 | | `0x0F33` | |
| [Replies](Capabilities-Replies.md#additions-to-the-tables) | 18 | 717, 718 | `0x0F20`–`0x0F24`, `0x0F2A`, `0x0F2E`–`0x0F30`, `0x0F32` | `0x0020` |
| [Reactions](Capabilities-Reactions.md#additions-to-the-tables) | 19 | 710–712 | `0x0F25`–`0x0F27`, `0x0F29`, `0x0F2B`–`0x0F2D`, `0x0F31` | `0x0021`, `0x0023` |
| [Server Emoji](Capabilities-Server-Emoji.md#additions-to-the-tables) | 20 | 713–716 | `0x0F38`–`0x0F42` | `0x0022` |
| Spare | | 719 | `0x0F28`, `0x0F34`–`0x0F37`, `0x0F43`–`0x0F4F` | `0x0024`–`0x002F` |

---

## Open Questions

1. **Inside Chat History, or beside it?** A live entry is Chat History's own layout. Would you rather it were that extension's: bit 4 sessions sent entries on 106 under a field in its `0x0F09`–`0x0F1F` block, with no bit of its own? This draft keeps it apart because a client may want live structure without history, and a server without history may still offer it.
2. **The ephemeral flag.** Is flag bit 3 the right place to say an ID is ephemeral, or would you rather a sub-field, leaving Chat History's flags alone?
3. **Ephemeral IDs.** Does numbering a private chat's lines in memory, and keeping what the drafts built on this one hold about them there until it closes, sit within the history extension's rule that private chats are never persisted? And is chat 0 the right way to number public lines a server does not keep?
4. **Matching an echo.** Order is enough until the server drops a line. Hotline-ng lets a client send its own ID with a line and see it on its echo alone; is that worth a field on 105 here?
5. **Line IDs and volume.** This draft keeps it a SHOULD, for the migration a MUST would cost ([Line IDs and Volume](#line-ids-and-volume)). Should a new server be told to number each channel's history apart from the start, so that the leak is only ever an older server's?
6. **The entry rules everywhere.** Should the three entry rules bind every Chat History server, not only one that confirms bit 17?
7. **Lines across links.** Should a ghost's structured line carry `DATA_LINK_LINE_ID` as a sub-field, so a client on two linked servers can tell it saw one line twice?
