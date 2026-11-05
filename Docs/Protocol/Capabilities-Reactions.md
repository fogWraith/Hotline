# Draft: Reactions

> **Status:** Draft proposal by Misha Nasledov (hxd-ng). **Not adopted**: nothing here is part of the protocol unless another document says so. It proposes capability bit 19, transactions 710–712, field IDs `0x0F25`–`0x0F27`, `0x0F29`, `0x0F2B`–`0x0F2D` and `0x0F31`, history sub-field types `0x0021` and `0x0023`, reason codes 3–5, and notification type 1, all within the blocks the [Structured Chat](Capabilities-Structured-Chat.md) draft proposes be kept for this family. It also asks for an access bit, React, from the extension allocation, without proposing its number. All of these were unassigned at the time of writing; none is allocated. Until they are, they are used only between servers and clients whose operators and users have chosen to try it, and nothing here changes what is asked of a server or client that has not.

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This extension lets a user attach an emoji to a message, and everyone who can read the message see the counts and who reacted. A reaction is a Unicode emoji, or, where the [Server Emoji](Capabilities-Server-Emoji.md) draft is also offered, one of the server's own. A message's author may ask to be told of reactions to it.

It builds on the [Structured Chat](Capabilities-Structured-Chat.md) draft, which gives chat lines the IDs a reaction names them by, and the [Replies](Capabilities-Replies.md) draft, which defines the targets a reaction names, the notification mechanism, the reason field and the reply shape. This document uses them by reference and does not repeat them. A server may offer reactions without replies: everything this draft takes from the Replies draft is confirmed by bit 19 as well, and bit 19 needs bit 17 beside it. For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md).

## Table of Contents

- [What Does Not Change](#what-does-not-change)
- [Negotiation](#negotiation)
  - [Capability Bit](#capability-bit)
  - [Access Bit](#access-bit)
  - [Server Configuration](#server-configuration)
  - [Login Reply](#login-reply)
- [Reaction Keys](#reaction-keys)
- [Who Reacts, and as Whom](#who-reacts-and-as-whom)
- [Send Reaction (710)](#send-reaction-710)
- [Reaction Changed (711)](#reaction-changed-711)
- [Get Reactions (712)](#get-reactions-712)
- [Chat History](#chat-history)
- [Notifications](#notifications)
- [Bounds](#bounds)
- [When the Target Goes Away](#when-the-target-goes-away)
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

- **A client that does not negotiate bit 19 never sees a reaction.** A server MUST NOT synthesize a "*** alice reacted 👍" line, or anything else, for it.
- **Everything a server already enforces still applies.** A reaction needs a permission of its own and read access to its target ([Who Reacts](#who-reacts-and-as-whom)), and costs what the server's spam accounting charges for it.
- The [Replies](Capabilities-Replies.md#what-does-not-change) draft's rules hold: the text the sender typed is never read or changed, a server MAY decorate only the formatted line of a reply for a session without bit 18, and private chats are never stored. A reaction is never shown that way: it decorates nothing.

---

## Negotiation

### Capability Bit

| Bit | Mask | Name | Description |
|---|---|---|---|
| 19 | `0x00080000` | `CAPABILITY_REACTIONS` | The client sends and understands reactions (710–712), and is sent private message references |

Bit 19 follows the [Replies](Capabilities-Replies.md#capability-bit) draft's rules for its bit: only beside bits 1 and 17, never on a server link, a statement of what the server can, sent in four bytes. A bit-19 session is sent private message references ([Private Message References](Capabilities-Replies.md#private-message-references)) whether or not bit 18 was confirmed. Bits 18 and 19 are independent. A session with bit 19 and not bit 20 is still sent the server-emoji keys in summaries, which it draws as a placeholder.

### Access Bit

One access bit is requested from the extension allocation. Bits 55–60 are taken, and this draft names no number it was not given.

| Name | Allows |
|---|---|
| `AccessReact` | Adding and removing one's own reactions, on every target kind |

A clear bit means "may not". It is granted as any bit is, by the server's account editor and administration tools. If it is allocated at 64 or above, only a session that negotiated `CAPABILITY_EXTENDED_PRIV` (bit 5) can read or edit it with the classic account transactions, and a server keeps it for every other editor as that extension says. Until a server's accounts record `AccessReact`, it MAY take its value from `ACCESS_SEND_CHAT` (10), as chat history falls back to `ACCESS_READ_CHAT`. That interim is the server's choice and visible as such in the bitmap it reports.

### Server Configuration

Recommended settings (implementation-defined):

| Setting | Type | Default | Description |
|---|---|---|---|
| `Reactions.Enabled` | bool | `false` | Accept and relay reactions |
| `Reactions.MaxKeysPerTarget` | int | `20` | Distinct reaction keys one message may carry; at most 255, since `DATA_REACTION_MINE` indexes them in a byte |
| `Reactions.MaxPerUserPerTarget` | int | `5` | Keys one account may have on one message at once |
| `Reactions.LiveWindow` | int | `200` | Each channel's most recent lines whose reaction changes are pushed live (711); older lines' are fetched on demand |
| `Reactions.RateBurst`, `Reactions.RatePerSecond` | int | `10`, `1` | Per-account reaction rate: a burst, then a steady rate |
| `Reactions.ReadOnlyChannels` | `allow` \| `deny` | `allow` | Whether a reader who may not post in a channel may react there. A server whose channels have settings MAY let each channel override it |

Reactions on ephemeral lines are held within the [Structured Chat](Capabilities-Structured-Chat.md#server-configuration) draft's `StructuredChat.EphemeralWindow` and `StructuredChat.EphemeralMaxBytes`.

### Login Reply

When bit 19 is confirmed, the login reply carries:

| Field | | Value |
|---|---|---|
| `DATA_REACTION_TARGETS` | MUST | The [target kinds](Capabilities-Replies.md#packed-target) this server accepts reactions on, as `1 << kind` |
| `DATA_MAX_REACTION_KEYS` | SHOULD | `Reactions.MaxKeysPerTarget` |
| `DATA_MAX_REACTIONS_PER_USER` | SHOULD | `Reactions.MaxPerUserPerTarget` |

A client MUST NOT send a kind `DATA_REACTION_TARGETS` leaves out, and a server that receives one in 710 or 712 refuses it `InvalidTarget`.

---

## Reaction Keys

`DATA_REACTION_KEY` is packed:

| Kind | Layout | Meaning |
|---|---|---|
| 1 Unicode | `u8 kind = 1`, then 1–64 bytes of UTF-8 | One emoji: exactly one extended grapheme cluster ([UAX #29](https://www.unicode.org/reports/tr29/)) |
| 2 Server emoji | `u8 kind = 2`, `u32 emoji_id` | One live entry of the server's catalog, by its ID, as the [Server Emoji](Capabilities-Server-Emoji.md#reactions) draft defines it. On a server that does not confirm bit 20, or from a session without it, refused `InvalidKey` |

A Unicode key:

- MUST be refused `InvalidKey` when it is empty, longer than 64 bytes, not valid UTF-8, more than one grapheme cluster, or holds a C0 or C1 control, whitespace, a private-use or surrogate code point, or a line or paragraph separator. Unassigned code points are not refused: a server's tables see an emoji newer than they are as unassigned, and a newer emoji must work on an older server.
- SHOULD be limited to a grapheme that contains an `Extended_Pictographic` character or is an RGI emoji sequence ([UTS #51](https://www.unicode.org/reports/tr51/)), and refused `InvalidKey` otherwise. The 64-byte cap bounds a grapheme that stacks combining marks. It is not tighter because the longest RGI sequences already run past 32 bytes: a kiss with two skin tones is 35.
- Is compared byte for byte. A client SHOULD send the fully-qualified form (with U+FE0F where the RGI set has one), so that U+2764 and U+2764 U+FE0F count together.

---

## Who Reacts, and as Whom

- A reaction belongs to an **account**, not a session or a nick. Every session of an account shares one set of reactions, and a reaction survives a nick change and a reconnect.
- **Only accounts react.** An account several people share, such as the guest account, cannot hold a reaction on behalf of one person, and everyone can see who reacted. A server MUST refuse a guest session's reactions with `NotAllowed`, whatever its access bits say.
- **Every target needs `AccessReact`** ([Access Bit](#access-bit)); without it, `NotAllowed`.
- A session that confirmed `CAPABILITY_MESSENGER_SESSION` (bit 8) is hidden from the classic world, and MUST NOT react to line or ephemeral line targets (`NotAllowed`).
- **Line targets:** the caller MUST be able to read the line: for public chat `ACCESS_READ_CHAT` (9), or the channel's own rules. A line older than the first one relayed live to this session can only have been read through history, and needs history's permission too (`AccessReadChatHistory`, 56, falling back to 9). Posting is not required: a reaction is not a line. A tombstoned line, one the caller cannot read, and one that does not exist are all `NoSuchTarget`, alike.
- **Read-only channels.** Where the caller may read a channel but not post in it, the reaction is allowed or refused `NotAllowed` by `Reactions.ReadOnlyChannels`, or by the channel's own setting where the server has one. The react bit and read access are needed either way.
- **Ephemeral line targets:** the caller MUST be a current member of the chat, or for chat 0 a reader of public chat, and the line MUST be among the chat's most recent `StructuredChat.EphemeralWindow` ephemeral lines and relayed while the caller was a member. Anything else is `NoSuchTarget`.
- **Line ID 0.** A line or ephemeral line target with line ID `0` names a line sent to one session ([Lines Sent to One Session](Capabilities-Structured-Chat.md#lines-sent-to-one-session)) and is `NoSuchTarget`.
- **Instant-message targets** are checked exactly as [IM Acknowledge (812)](Capabilities-Messaging.md#im-acknowledge-812) is. The request names the conversation's other party in `DATA_FRIEND_LOGIN`, beside the target's own sender Login and GUID. The server checks that the caller and that party are friends or have an approval ([message requests draft](Capabilities-Messaging-Requests.md)), and that neither has blocked the other, and relays the reaction to that party. It never keeps or looks up the message: the party's client matches the GUID it holds, and a reaction to a GUID nobody holds is noise to a friend, nothing more. No relationship, and a block, are both `NoSuchTarget`. An instant message is an instant-message target wherever the server keeps it, in its offline queue or in a mailbox beside private messages, and never a stored private message target: a reaction naming it by a stored reference is dropped like one naming nothing, so that one message has one set of reactions. A server that lists instant messages among stored ones MUST mark each and give its GUID, as the [Hotline-ng binding](#hotline-ng-binding) does.
- **Stored private message targets** (a durable reference, stored bit set): the request carries the target alone. The server looks the message up by its reference among the messages the caller sent or received, within the caller's view ([When the Target Goes Away](#when-the-target-goes-away)). If it finds one, it applies the reaction, keeps it with the message, and tells the parties as [Reaction Changed](#reaction-changed-711) says. If it finds none, because the message is outside the caller's view, is someone else's or never existed, or because the other party has blocked the caller, it drops the reaction silently. **The reply is identical either way**: the same success, with no count, in the same shape, and a server SHOULD do the same work on both paths, so that timing is the only side channel left.
- **Unstored private message targets** (an ephemeral reference): the request names the other party's session in `FieldUserID`, as a 108 does. The server checks only that the caller could send that session a 108, and relays the reaction to it, held nowhere, never checking the reference: the party's client resolves it from what it was sent, and a reaction to a reference it does not hold is noise. A server that would refuse the 108, for a block or otherwise, refuses the reaction with `NoSuchTarget`. Hotline-ng has no ephemeral private message references yet.
- A user MAY react to their own message.
- **Order.** A reaction naming a line above the channel's [relay watermark](Capabilities-Replies.md#performance-considerations) waits until the line is relayed, or is answered `NoSuchTarget`.

---

## Send Reaction (710)

**Client → Server (request/reply)**

| Field | Type | Required | Description |
|---|---|---|---|
| `DATA_TARGET` | Binary | Yes | The [target](Capabilities-Replies.md#packed-target) |
| `DATA_FRIEND_LOGIN` (`0x0600`) | String | Instant-message targets | The conversation's other party |
| `FieldUserID` (103) | UInt16 | Unstored private message targets | The other party's session |
| `DATA_REACTION_KEY` | Binary | Yes, but for action 3 | The [key](#reaction-keys) |
| `DATA_REACTION_ACTION` | UInt16 | Yes | `1` add, `2` remove, `3` clear the target (moderators only) |

**Reply fields:** on success for a line or ephemeral line target, `DATA_REACTION_COUNT`, the key's count after the act; for a private message target, nothing beyond success, whatever became of the reaction. Refusals use the [reply shape](Capabilities-Replies.md#reply-shape) with `DATA_CHAT_REASON`. A server MAY commit reactions in groups, and MAY reply before the reaction is durable.

- Adding a key the caller already holds, or removing one it does not, succeeds, changes nothing and announces nothing.
- Action 3 removes every reaction on a line or ephemeral line target. It is a moderation act: a server SHOULD allow it only to an account that may remove other users' content there, for example one holding `ACCESS_DISCONNECT_USER` (22), refuse it `NotAllowed` otherwise, and record it as it records its other moderation.
- A server MUST check the action against the [bounds](#bounds) before it is applied. For a stored private message target, an act past `TooManyKeys` or `TooManyOfYours` is dropped silently and answered as success, like any other drop, since refusing it would show that the message exists. Checks that need no lookup, `InvalidKey` and `RateLimited`, are answered as for any target.

---

## Reaction Changed (711)

**Server → Client (notification)**

A 711 carries one or more groups, each beginning with `DATA_TARGET`, in 712's reply shape:

| Group | Fields after the target | For |
|---|---|---|
| Summary | `DATA_REACTION_ACTION` = `4`, `DATA_REACTION_SUMMARY`, then `DATA_REACTION_MINE` when the recipient's account holds keys on it | Line, ephemeral line and stored private message targets |
| Cleared | `DATA_REACTION_ACTION` = `3` | Line and ephemeral line targets |
| Change | `DATA_REACTION_ACTION` = `1` or `2`, `DATA_REACTION_KEY`, and `DATA_FRIEND_LOGIN` for an instant message | Instant-message and unstored private message targets |

**Line and ephemeral line targets** are told as summaries, to every session that negotiated bit 19 and can read the target: for a line, every session that reads its channel, and for an ephemeral line, the chat's current members who were sent it. A summary names no reactor; who reacted is Get Reactions' to say. That keeps a summary the same for every recipient but the last field.

- **Batching.** A server SHOULD gather a channel's changes over a window of at least 1 second and send each session at most one 711 per channel per window, a group per changed target.
- **Latest wins.** A summary reflects the target when it is sent. A server MAY replace a summary still queued unsent to a session with a newer one for the same target, so a session's backlog of reaction pushes is at most `Reactions.LiveWindow` entries per channel and never drives it past its queue's cap. A pending summary for a target since tombstoned is discarded, and one for a target since cleared becomes a cleared group.
- **The live window.** A server MAY push only for each channel's most recent `Reactions.LiveWindow` lines, and leave older lines to Get Reactions. A client SHOULD refresh older lines' reactions with 712 when it shows them.
- **Order.** Groups for one target are sent in the order their changes were applied.

A client keeps a target's reactions from Get Chat History or Get Reactions and replaces them with each summary. After a reconnect it asks again.

**Ephemeral line targets** are counted in memory for the chat's most recent `StructuredChat.EphemeralWindow` ephemeral lines, within `StructuredChat.EphemeralMaxBytes`. A line that leaves the window takes its reactions with it, unannounced, and everything is dropped when the chat closes or the server restarts. There is no summary to fetch later: a member who joins later sees no earlier reactions.

**Stored private message targets** are kept with the message, so their summary survives a logoff and reaches every device. A change is told as a summary to the live sessions of both parties that negotiated bit 19, other than the reactor's own: to the recipient's once the message has been delivered to it, and to the sender's only while the sender's view lasts ([When the Target Goes Away](#when-the-target-goes-away)). A dropped reaction is told to nobody. A client that sees the message later reads its summary from Get Reactions, or, on Hotline-ng, from the inbox row.

**Unstored private message targets** are relayed as change groups and held nowhere, as a live read receipt is. A reaction goes to the session named, if it is still connected and negotiated bit 19, and is otherwise dropped.

**Instant-message targets** are relayed as change groups to the party named, as receipts are, naming the reactor by `DATA_FRIEND_LOGIN` alone. When that party has no live session, the server SHOULD keep the latest state of each (reactor, message, key), at most 100 per recipient, the oldest evicted first, and none older than the offline queue keeps messages. It delivers them at the party's next login, after Get Roster, as one 711 per conversation.

A reaction carries no presence. A reaction to an instant message reveals activity as a receipt does, and nothing in a 711 says whether its reactor is online, away or `Invisible`.

---

## Get Reactions (712)

**Client → Server (request/reply)**

Refreshes what a client holds, and says who reacted. Who reacted is shown to everyone who can read the target.

| Field | Type | Required | Description |
|---|---|---|---|
| `DATA_TARGET` | Binary | Yes, repeated, at most 50 | Line targets, ephemeral line targets in the window, and stored private message targets. An instant-message target, or an unstored private message target, is refused `InvalidTarget` |
| `DATA_REACTION_KEY` | Binary | No | With exactly one target: list who holds this key |

More than 50 targets is refused `TooManyTargets`. Refusals use the [reply shape](Capabilities-Replies.md#reply-shape) with `DATA_CHAT_REASON`. A server SHOULD price 712 in its spam points as 1 plus 1 for every 10 targets, rounding up.

**Reply fields:** for each target the caller can read, in request order, `DATA_TARGET`, one `DATA_REACTION_SUMMARY` and, when the caller's account holds keys on it, one `DATA_REACTION_MINE`. A target the caller cannot read, a tombstoned line, and one that does not exist are left out alike, under the same read rules as Send Reaction, history's permission included. A stored private message target is always answered, with the summary when the caller is a party whose view includes it ([When the Target Goes Away](#when-the-target-goes-away)), and with an empty summary otherwise: anyone else, a party whose view has ended included, sees nothing, in the same shape a party sees for a message with no reactions. With `DATA_REACTION_KEY`, the group is followed by one `FieldUserName` (102) per reactor, newest first, at most 100, each the nick the reactor had when they reacted. A server MUST answer it for any reader of the target, and for a stored private message only within the caller's view. Logins are never sent.

A summary has two parts, so that the shared part can be built once and cached:

- `DATA_REACTION_SUMMARY`, the same for every reader: `u16 n`, then `n` entries of `u32 count`, `u16 key_len`, `key` (a packed `DATA_REACTION_KEY`, its kind byte included), in the order the keys were first added. It includes server-emoji keys for every reader; one without bit 20 draws them as a placeholder.
- `DATA_REACTION_MINE`, the reader's own: `u8 n`, then `n` `u8` indexes into that summary's entries, the keys the reader's account holds. It always comes last, so a server appends it to the shared bytes it cached.

---

## Chat History

A Get Chat History (700) entry carries, as optional sub-fields in the family's range ([Structured Chat](Capabilities-Structured-Chat.md#chat-history), which also gives their audiences, the tombstone rule and the reply-size rule):

| Sub-Type | Name | Sent to | Content |
|---|---|---|---|
| `0x0021` | `REACTIONS` | Bit 19 | The line's shared `DATA_REACTION_SUMMARY`, omitted when the line has none |
| `0x0023` | `REACTIONS_MINE` | Bit 19 | This reader's `DATA_REACTION_MINE` for the line, omitted when empty |

---

## Notifications

This draft defines notification type 1 of the [Replies](Capabilities-Replies.md#notifications) draft's mechanism:

| Type bit | Name | Told when | To sessions with |
|---|---|---|---|
| 1 | `Reactions` | Someone else adds a reaction to one of the author's lines | Bit 19 |

A Reactions notification (718) carries `DATA_REACTION_KEY` when it stands for one reaction. A server SHOULD coalesce reaction notifications per target per author within a window, sending one 718 with `DATA_NOTIFY_COUNT` for the number of reactions it stands for ("5 reactions") and no key or reactor, even below the per-author bound the Replies draft sets.

---

## Bounds

- A target carries at most `Reactions.MaxKeysPerTarget` distinct keys. Adding a new key past it is refused `TooManyKeys`; adding to an existing key never is.
- One account holds at most `Reactions.MaxPerUserPerTarget` keys on one target; past it, `TooManyOfYours`.
- **Rate.** A server MUST rate-limit Send Reaction per account, by default a burst of `Reactions.RateBurst` (10) and then `Reactions.RatePerSecond` (1) a second, and answer `RateLimited` past it. It MAY hold an address to a ceiling as well, set loosely, since many people can share one address behind carrier-grade NAT. It SHOULD price 710 in its spam points like other requests. A reaction is not a chat line and is not counted by chat flood rules.
- Reaction state for ephemeral lines counts toward `StructuredChat.EphemeralMaxBytes`, trimmed as the [Structured Chat](Capabilities-Structured-Chat.md#ephemeral-memory) draft says.

---

## When the Target Goes Away

- **A line removed by a moderator** (tombstoned) loses its reactions, and history shows the tombstone with none.
- **A line pruned by retention** takes its reactions with it.
- **An ephemeral line** loses its reactions when it leaves the window, and all of them go when the chat closes or the server restarts.
- **A stored private message** is seen by each party for a time of its own. The sender's view, its reactions and the right to add them, lasts from when the message was sent until the server's retention for an unread message has passed, by the server's clock, whatever its pruning has yet to do. Nothing the recipient does moves it: a server that deletes the message sooner, because it was read or at the recipient's request, MUST keep its reference, its parties, when it was sent, its reactions and, where it deduplicates resends, what it matches them by, until the sender's view ends, and answer the sender from them as from the message. A server that deduplicates resends of a private message, as the Hotline-ng binding's `msg` does by `guid`, MUST answer a resend the same whether or not the first was read or deleted. Lengthening the retention for unread messages reopens the sender's view of messages already past its old end, and with it the difference between those that were read and those that were not. The recipient's view lasts as long as the message is kept for it, and MAY last until the sender's ends. Once the sender's view has ended, the sender is answered as a non-party and told nothing more of the message; a reaction the recipient adds then is the recipient's alone. A message deleted with an account takes its reactions with it.
- **A user's reactions** go when their account is deleted.
- **A purge of a person** removes their reactions within the purge's own scope: a purge of the last hour's content removes the reactions they made in that hour, and a purge of everything removes them all. It is announced only for targets inside each channel's live window, as summaries in that channel's next batch; older lines are refreshed when a client shows them. For a stored private message, it is announced as [Reaction Changed](#reaction-changed-711) says for one.
- **A server emoji removed from the catalog** stays in the reactions that use it, as the [Server Emoji](Capabilities-Server-Emoji.md#manage-emoji-716) draft says.

---

## Text Encoding

Bit 19 requires `CAPABILITY_TEXT_ENCODING` (bit 1). Reaction keys are **Binary** fields whose text is UTF-8 by definition, never transcoded. A Mac Roman session cannot send or receive a reaction: a Unicode reaction has no Mac Roman form, and a shortcode in its place would be a second spelling of every key.

---

## Across Server Links

Reactions stay on the server where they were made. Users there may react to a linked user's line where the server keeps it in its own history; the line's author and its home server never learn of it, and a linked author is never notified. `DATA_LINK_LINE_ID` names a line across the network and was assigned so that a later version could name lines; a later version of this draft could carry reactions over it.

---

## Data Objects

Within the family's block ([Structured Chat](Capabilities-Structured-Chat.md#data-objects)). `DATA_TARGET`, `DATA_CHAT_REASON` and the notification fields this draft uses are the [Replies](Capabilities-Replies.md#data-objects) draft's. Integers are big-endian.

| ID (hex) | Decimal | Name | Type | Description |
|---|---|---|---|---|
| `0x0F25` | 3877 | `DATA_REACTION_KEY` | Binary | A [reaction key](#reaction-keys) |
| `0x0F26` | 3878 | `DATA_REACTION_ACTION` | UInt16 | 1 add, 2 remove, 3 clear (or cleared), 4 summary |
| `0x0F27` | 3879 | `DATA_REACTION_COUNT` | UInt32 | A key's count on a target |
| `0x0F29` | 3881 | `DATA_REACTION_SUMMARY` | Binary | Packed keys and counts for one target, shared by every reader |
| `0x0F2B` | 3883 | `DATA_REACTION_TARGETS` | UInt16 | Login reply: target kinds accepted, as `DATA_REPLY_TARGETS` |
| `0x0F2C` | 3884 | `DATA_MAX_REACTION_KEYS` | UInt32 | Login reply: `Reactions.MaxKeysPerTarget` |
| `0x0F2D` | 3885 | `DATA_MAX_REACTIONS_PER_USER` | UInt32 | Login reply: `Reactions.MaxPerUserPerTarget` |
| `0x0F31` | 3889 | `DATA_REACTION_MINE` | Binary | The reader's own keys, as indexes into a summary; always last in its group |

---

## Reason Codes

The family's enumeration is the [Replies](Capabilities-Replies.md#reason-codes) draft's. This draft adds:

| Code | Name | Meaning |
|---|---|---|
| 3 | `InvalidKey` | A key outside [Reaction Keys](#reaction-keys)' rules |
| 4 | `TooManyKeys` | The target is at `MaxKeysPerTarget` |
| 5 | `TooManyOfYours` | The caller is at `MaxPerUserPerTarget` on this target |

---

## Performance Considerations

What an implementer must watch, beyond the bounds above:

- **Fan-out.** A reaction is one small change heard by everyone reading the channel. Batch per channel per session, at most one 711 a second, carrying summaries rather than changes, and only for the live window's lines ([Reaction Changed](#reaction-changed-711)). Replacing a queued summary with a newer one keeps a slow reader's backlog bounded by the window, not by the reaction rate.
- **Encode once.** A summary's shared part is the same for every reader. Build it once per line or per page, and append only `DATA_REACTION_MINE` per reader.
- **Batch reads.** A history page's summaries and the reader's own keys are two queries over the page's IDs, not one per line.
- **Indexes.** Reactions keyed by target, for summaries, redaction and retention; by target, key and time descending, for who reacted; and by reactor and time, for scoped purges. Keep the reactor's nick on the row, so who-reacted needs no join. A reaction to a stored private message costs one lookup, of the message or of what the server keeps of it until the sender's view ends, by its reference, which needs a unique index on references; the not-found path SHOULD cost the same.
- **Commits.** Reactions MAY be committed in groups, and the 710 reply MAY precede durability. The author lookup for notifications happens after commit ([Replies](Capabilities-Replies.md#performance-considerations)).
- **Ordering.** Changes to one target are applied and emitted in order; a lock per target suffices. Nothing about a line goes out before the line ([relay watermark](Capabilities-Replies.md#performance-considerations)).
- **Notifications** coalesce per target per author.

---

## Security Considerations

- **Reactions are public to the target's readers.** A count says how many accounts reacted, and Get Reactions says who, by nick. Neither ever carries a Login, and guests cannot react, so every reactor is a person the server can name.
- **A target is not a probe.** A line the caller cannot read answers `NoSuchTarget` exactly as a removed or missing one, and Get Reactions leaves all three out. A stored private message target answers the same whether the reaction was applied or dropped: the message outside the reactor's view, someone else's, never sent, or its other party blocking the reactor. There is no existence oracle, for the same reason replies never check targets: answering by whether a message still exists would let a sender learn whether it was read or deleted. That is why the sender's view is fixed by when the message was sent: while it lasts, nothing the sender is answered or told (Send Reaction, Get Reactions, 711, or a resend the server deduplicates, such as a Hotline-ng `msg` under the same `guid`) depends on whether the message is still kept, and after it ends the sender is answered as a non-party whatever became of it. An unstored one is never looked up at all. An instant-message reaction goes only to a party the reactor may message, under 812's checks, so naming a GUID proves nothing and reaches no stranger.
- **Presence stays hidden.** A 711 carries no presence and names no reactor of a line at all; an instant-message reactor is named by Login alone.
- **What the server sees.** A reaction to a sealed message tells the server which message was reacted to, though not what it said.
- **No image of its own.** A reaction is a Unicode emoji or a catalog entry's ID, never an image.
- **Bounded input.** Keys are at most 64 bytes, and 712 names at most 50 targets.
- **Abuse.** Reactions need their own bit, cost spam points, and are bounded per target and per user. A moderator can clear a target.

---

## Hotline-ng Binding

*Implementers of the classic extension can skip to [Additions to the Tables](#additions-to-the-tables).*

For a server that also speaks the [Hotline-ng](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng.md) JSON protocol, the same feature as capability `reactions`, under the same rules and on the [Replies](Capabilities-Replies.md#hotline-ng-binding) draft's targets. One store serves both wires: a reaction made on one is counted and announced on the other.

| `req` | params | ok | errors | weight |
|---|---|---|---|---|
| `react` | `target`, `key`, `remove?` (default `false`), and `with` (the other party's login) for an `im` target; or `target` and `clear: true` | `{ "count"? }` | `no_such_target`, `access_denied` (no react bit, a guest, a hidden messenger, a read-only channel that denies, a clear without the right), `bad_request` (an invalid key or target), `too_many_reactions`, `rate_limited`, `no_such_emoji` | a write |
| `reactions` | `targets` (1–50, line and `pm` targets), `key?` with exactly one target | `{ "reactions": [ { "target", "keys": [ { "key", "count" } ], "mine"?: [ indexes ], "from"?: [ { "nick" } ] } ] }` | `bad_request` (an instant-message target, or more than 50) | 1 + ceil(targets / 10) |

A key is `{ "emoji": "👍" }`, or `{ "custom": 1234 }` with [Server Emoji](Capabilities-Server-Emoji.md#hotline-ng-binding). Event `reactions_changed`: `{ "changes": [ { "target", "action": "summary", "keys", "mine"? } | { "target", "action": "clear" } | { "target", "action": "add" | "remove", "key", "from": { "login" } } ] }`, batched per room as 711 is. A `history` line carries `reactions: [ { "key", "count" } ]`, the shared part, and `reactions_mine: [ indexes ]`, the reader's. An `inbox` row carries the same two for a stored private message, and so does a `msg` event that delivers one that waited; a `msg` delivered live carries neither. An instant message's `inbox` row, and a `msg` that hands one over as mail, carry `im: true` and its `guid`, which with `from.login` make the `im` target that reacts to it. A `pm` target takes the target alone, and `react` answers it `{}` whether the reaction was applied or dropped. After `resync_required`, a client asks `reactions` again for the targets it shows, beside its `history` catch-up. The login block `reactions`: `{ "targets": ["line", "im", "pm"], "max_keys": 20, "max_per_user": 5 }`. The `reactions` notification type is the Replies binding's `notify`.

---

## Additions to the Tables

All **proposed, pending allocation**, within the blocks the [Structured Chat](Capabilities-Structured-Chat.md#additions-to-the-tables) draft proposes be kept for this family.

Capability bit, as a row of [Capabilities.md](Capabilities.md#defined-capability-bits):

| Bit | Mask | Name | Description |
|---|---|---|---|
| 19 | `0x00080000` | `CAPABILITY_REACTIONS` | **Draft**, not adopted. Emoji reactions on chat lines and messages: Send Reaction (710), Reaction Changed (711), Get Reactions (712), on the [Replies](Capabilities-Replies.md) draft's targets. Only alongside bits 1 and 17, and never on a server link (bit 11). |

Transactions:

| ID | Name | Type | Description |
|---|---|---|---|
| 710 (`0x02C6`) | Send Reaction | Request/reply | Add, remove or clear reactions on a target |
| 711 (`0x02C7`) | Reaction Changed | Server → client | Batched summaries and changes |
| 712 (`0x02C8`) | Get Reactions | Request/reply | Summaries for up to 50 targets, and who reacted |

Fields: `0x0F25`–`0x0F27`, `0x0F29`, `0x0F2B`–`0x0F2D` and `0x0F31`, as in [Data Objects](#data-objects).

Chat history sub-fields `0x0021` `REACTIONS` and `0x0023` `REACTIONS_MINE`. Reason codes 3–5, in the family's enumeration. Notification type 1.

Access bit: `AccessReact`, **requested**. Bits 55–60 are taken, and no number is named here.

---

## Open Questions

1. **The access bit.** Which number would you give `AccessReact`, and is a server's interim fallback from `ACCESS_SEND_CHAT` acceptable?
2. **Unicode keys.** Is "one pictographic grapheme, byte-for-byte comparison" enough, or should servers canonicalize the variation selector so U+2764 and U+2764 U+FE0F always count together?
3. **Telling a classic author.** A client without bit 19 is never shown a reaction in the room, and that stays. Should a server be allowed an optional notice, off by default, to the line's author alone, coalesced per window ("3 reactions to your line '…': 👍 2, 🎉 1")? Emoji are unmappable in Mac Roman, so such a notice would have to name them in words, or be offered only to UTF-8 sessions.
4. **The sender's view.** Is ending it at the unread retention right, and should the login reply say how long it is, so that a client stops offering to react once it has passed?
