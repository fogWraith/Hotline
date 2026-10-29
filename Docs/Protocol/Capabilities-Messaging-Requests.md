# Draft: Message Requests for the Instant Messaging Extension

> **Status:** Draft proposal by Misha Nasledov (hxd-ng). **Not adopted**: nothing here is part of the [Instant Messaging Extension](Capabilities-Messaging.md) unless that document says so. It proposes reason code `Requested`, transactions Get Message Requests, Answer Message Request, Clear Message Requests and Message Requests Changed, one limits field and a few general fields. The numbers below are placeholders (`TBD`); I ask that they be assigned, from 829–839 for the transactions. The general fields, with those of the identity draft's version 2, overflow the four IDs `0x062C`–`0x062F`, so I suggest placing both drafts' fields together in a new messaging block, from `0x0650`, and keeping `0x062C`–`0x062F` free. Until it is adopted, they are used only between servers and clients whose operators and users have chosen to try it, and nothing here changes what is asked of a server or client that has not.

An optional addition to the [Instant Messaging Extension](Capabilities-Messaging.md), written in that document's format and linking to its sections.

Today a message to someone who is not a friend is refused `NotFriends`, so the only way to reach a stranger is a friend request and its one-line note. This draft lets such a message wait, outside the recipient's offline queue, as a **message request**, until the recipient approves the conversation, adds the sender as a friend, deletes it, or blocks the sender. It also lets a server bound friend requests, which today a stranger can send without limit.

A **request** is one sender's waiting messages to one recipient; counts of requests below count senders.

---

## Message Requests

### What Does Not Change

- **Everything here is opt-in, per session.** A session that never sends [Get Message Requests](#get-message-requests-tbd) is answered as the extension answers it today, whatever its account's other sessions have done, is sent nothing it has not seen before, and never approves anything by writing. Requests pass only from a session that has listed requests to an account one of whose clients has.
- **What an existing client can still notice** is written as what a server MAY do, with replies the extension already has:
  - an approval made from another of its account's sessions counts toward `MaxRosterSize`, so an 802 can be refused `RosterFull` below the roster the client shows ([Approvals](#approvals));
  - the [friend request bounds](#friend-request-bounds): an 802 answered `RateLimited`; a held or dropped 802 that reads as one never answered; an 805 refused `RosterFull` below the cap the client sees; a request that lapses, which reads to its sender as a decline and to its recipient as a withdrawal; and two crossing requests that accept each other.
- **Friends are unaffected.** Messages between accepted friends, their offline queue and their receipts work exactly as before.
- **The reserved logins and `AccessMessaging` rules apply unchanged.** Both parties to a request hold `AccessMessaging`, and an account the server refuses for messaging neither sends nor receives one.

### Taking Requests

An account **takes requests** once a client of it has listed them with Get Message Requests, unless it has turned them off with `DATA_TAKES_REQUESTS` = `0` in [Set Presence (808)](Capabilities-Messaging.md#set-presence-808), beside `DATA_DISCOVERABLE`. Listing is the signal because it is the one thing a client does only if it can show requests: a user whose clients have never shown requests is never sent one. The sender's side is the session's own: a session that has not listed requests makes none and is never told `Requested`.

A server that implements this draft and has it enabled sends `DATA_MAX_REQUEST_MESSAGES` in the login reply, beside the extension's [Server Limits Advertisement](Capabilities-Messaging.md#server-limits-advertisement). Its absence means the server takes no requests: a client offers no requests view and expects `NotFriends`. This is the same convention as `DATA_MAX_ICON_BYTES` for buddy icons.

### What Becomes a Request

An [IM Send (810)](Capabilities-Messaging.md#im-send-810) from a session that has listed requests, to a Login that is not an accepted friend of the caller, is answered in this order:

0. **No such Login, or one the server refuses for messaging:** as [Find User (822)](Capabilities-Messaging.md#find-user-822) answers it. These sends count toward Find User's rate limit, since they now say whether a Login exists.
1. **The recipient has blocked the caller:** `AccountNotFound`, before any bound is checked, as Find User answers it, carrying to 810 the extension's SHOULD for [Add Friend (802)](Capabilities-Messaging.md#add-friend-802).
2. **The caller has blocked the recipient, or the recipient does not take requests:** `NotFriends`, as today.
3. **The recipient has approved the caller** ([Approvals](#approvals)): an IM, answered `OfflineQueued`.
4. **Otherwise:** a request, answered `Requested`, and stored, held or dropped by the [bounds](#bounds).

A request carries text only. It may carry the identity draft's signed form, `DATA_MESSAGE_SIGNED`, under that draft's rules for a signed message, which apply to a request as to an IM. A sealed message to anyone but an accepted friend is refused `NotFriends`, so every request can be read, and reported, by its recipient. A block also deletes the blocker's own requests to the blocked account, so the blocked account never answers one and meets the block.

An account that once took requests but has not listed them within `Messaging.RequestRetentionDays` SHOULD still be answered as taking them, with the request dropped. Answering `NotFriends` instead would tell a stranger when the recipient last used a client that shows requests.

### What the Sender Sees

`Requested` is a success, like `OfflineQueued`, and is the same answer whether the request was stored, held or dropped, and whether the recipient is online, has looked, or never will. Until the recipient acts, the sender gets no receipts, no typing and no presence from the recipient: the server drops [IM Acknowledge (812)](Capabilities-Messaging.md#im-acknowledge-812) and [IM Typing (813)](Capabilities-Messaging.md#im-typing-813) between two accounts with no friendship or approval.

A retry under a `DATA_MESSAGE_GUID` already sent is answered as the original was, and is not counted again.

### What the Recipient Does

A request is not delivered as an [IM Deliver (811)](Capabilities-Messaging.md#im-deliver-811), is not part of the [offline queue](Capabilities-Messaging.md#offline-delivery) and does not count toward `MaxOfflinePerRecipient`. Sessions that have listed requests are told whenever their listing would change, by [Message Requests Changed](#message-requests-changed-tbd). For each sender, the recipient may:

- **Approve the conversation** ([Approvals](#approvals)). The waiting messages are delivered as IMs. Replying with 810, from a session that has listed requests, to a sender whose request is waiting approves it too: writing back is consent.
- **Add the sender as a friend**, with [Add Friend (802)](Capabilities-Messaging.md#add-friend-802). The request stays listed until the sender accepts, when its messages are delivered.
- **Delete the request.** Its messages are deleted, the sender is told nothing, and the sender is [held](#holds).
- **Block the sender**, with [Block User (806)](Capabilities-Messaging.md#block-user-806), which also deletes the request.

**Clearing in bulk.** [Clear Message Requests](#clear-message-requests-tbd) declines every friend request and deletes every message request received within an optional span, so that a recipient who has been flooded can clear the flood and keep what came before it. A message request falls in the span by its first message. Each sender is held, and told, as one decline or delete would hold and tell it. Approvals, friendships and the identity draft's offers on an existing friendship are untouched.

### Approvals

An **approval** is a recipient's permission for one non-friend to message them. It is not a roster entry and is never reported in [Get Roster (800)](Capabilities-Messaging.md#get-roster-800) or [Roster Entry (801)](Capabilities-Messaging.md#roster-entry-801): it is listed with the requests, to both parties, so the approved party has a row for the messages it receives. It counts toward the approver's `MaxRosterSize`, and past it the approval, or a reply that would make one, is refused `RosterFull` and nothing is stored.

Messages pass both ways under an approval, but it shows less than a friendship:

- **Every send is answered `OfflineQueued`**, whether or not a live session took it, and `Delivered` receipts are not forwarded. `Read` receipts are, since a read is a person's act. Typing is not sent. An approved stranger learns no more of when the recipient is around than the replies and the read receipts say.
- **IMs and `Read` receipts only.** Presence, profiles, capabilities, file offers (814) and calls (818) stay the friends', under [User Discovery](Capabilities-Messaging.md#user-discovery) and the profiles' [Visibility](Capabilities-Messaging.md#visibility).
- **Only to sessions that list requests.** Its 811 and `Read` 812 go to each party's sessions that have listed requests, and wait for one otherwise, since a client that does not know approvals has no row to show them under. Get Message Requests delivers what waits for the caller under its approvals, after its reply.

Messages waiting under an approval are delivered only while it lasts. Either party's Block User (806) ends an approval, as does a friendship between the pair, which then takes the approval's waiting messages. The approver ends one with [Answer Message Request](#answer-message-request-tbd), which holds the sender as a delete does. [Remove Friend (803)](Capabilities-Messaging.md#remove-friend-803) does not name an approval, since the sender has no roster row.

### Bounds

Requests are the one way a stranger can put something in front of a user, so a server bounds them more tightly than the offline queue. Every bound counts what the sender **sent**, whatever became of it, so that what the recipient did never changes what the sender is told. A server SHOULD also make the answer take the same time whatever became of the request.

- **Per sender and recipient:** at most `Messaging.RequestMessages` messages in a request to one recipient within `Messaging.RequestRetentionDays`. Past it the sender is refused `QueueFull`.
- **Per sender:** at most `Messaging.RequestsPerDay` distinct recipients in a rolling day, refused `RateLimited`. An 810 and an 802 to one stranger are one recipient.
- **Per address:** a server MAY apply a looser ceiling, `Messaging.RequestsPerAddress`, to the distinct recipients requested from one network address in a rolling day, so that new accounts are not new budgets, refused `RateLimited`. It should be loose, since many people can share one address behind carrier-grade NAT.
- **Per recipient:** at most `Messaging.MaxRequests` senders waiting. Past it a new request is dropped, answered and counted as if stored, so that a burst of new accounts cannot flush out the requests already waiting. The listing says how many were dropped.
- **Retention:** a request not acted on is deleted after `Messaging.RequestRetentionDays`.

#### Holds

After the recipient deletes a sender's request, declines its friend request, removes it as a friend or ends its approval, that sender's requests to the recipient are **held** for `Messaging.RequestRetentionDays`, and its friend requests too where the server bounds them ([below](#friend-request-bounds)): answered as any is, counted, and stored nowhere the recipient can see. Removing someone has to stop them contacting you, which is the extension's own reason for [mutual removal](Capabilities-Messaging.md#remove-friend-803).

#### Friend Request Bounds

A server MAY bound friend requests the same way, with replies the extension already has. [Add Friend (802)](Capabilities-Messaging.md#add-friend-802) shares the per-sender and per-address bounds and the hold, and:

- **Per recipient:** at most `Messaging.MaxFriendRequests` delivered and waiting. Past it, or from a held sender, an 802 is answered `PendingOut` as usual and keeps the sender's `PendingOut`, so a repeat is answered `RequestPending` as any is; the recipient is sent no [Friend Request (804)](Capabilities-Messaging.md#friend-request-804), never sees it as `PendingIn`, and never sees its note. The listing counts these among the dropped.
- **Not in the recipient's roster size:** a pending request counts toward its sender's `MaxRosterSize` alone, so that strangers cannot fill a roster and leave its owner unable to add a friend. A server that does this also applies `Messaging.MaxFriendRequests`. A [Friend Response (805)](Capabilities-Messaging.md#friend-response-805) that would accept past the recipient's cap is refused `RosterFull`, and the request stays as it was.
- **Lapse:** a friend request not answered within `Messaging.RequestRetentionDays` is dropped, and is absent from both parties' next Get Roster.
- **Crossing:** an 802 to someone whose own request to the caller is waiting, delivered or not, accepts that request as an 805 would, unless either carries the identity draft's offer. Its reply carries `DATA_ROSTER_STATE` = `Accepted`, and both parties receive the 801s an acceptance sends.

### Listing

[Get Message Requests](#get-message-requests-tbd) lists requests by sender, oldest first, then the approvals, and says how many requests and friend requests were dropped since the last listing. Each sender carries `DATA_FRIEND_IDENTITY` where the [identity draft](Capabilities-Messaging-Identity.md) would show it, or where a request message is signed, since its certificate names the identity anyway. Each message carries its `DATA_MESSAGE_GUID`, which with the sender identifies it, so that a server that supports reporting can let the recipient report it. A client that shows requests lists them once after each login, since listing is also what marks a session as one to send approvals and Message Requests Changed to.

### Accounts

Deletion, rename and the revocation of `AccessMessaging` treat requests, approvals, holds and the counts behind the bounds as the extension's [Persistence](Capabilities-Messaging.md#persistence) treats roster rows. A Login registered again inherits none of them.

### Server Configuration

| Setting | Type | Default | Description |
|---|---|---|---|
| `Messaging.Requests` | bool | `false` | Whether the server takes requests at all |
| `Messaging.RequestMessages` | int | `3` | Messages one sender may send one recipient as a request within the retention period; strict on purpose, since a stranger's first contact is the one place a server cannot tell spam from a person |
| `Messaging.RequestsPerDay` | int | `20` | Distinct recipients one sender may request in a rolling day |
| `Messaging.RequestsPerAddress` | int | `60` | Distinct recipients requested from one address in a rolling day; `0` = no such bound |
| `Messaging.MaxRequests` | int | `100` | Senders waiting per recipient |
| `Messaging.MaxFriendRequests` | int | `100` | Delivered friend requests waiting per recipient |
| `Messaging.RequestRetentionDays` | int | `30` | How long a request, a hold and an unanswered friend request last |

---

## Transactions

### Get Message Requests (TBD)

Request/reply. Lists the caller's requests and approvals, paged.

**Request fields:** `DATA_REQUEST_CURSOR` (optional; absent for the first page).

**Reply fields:** first, on the first page, `DATA_TAKES_REQUESTS`, the caller's own setting, and `DATA_REQUEST_COUNT`, the requests and friend requests dropped since the last listing, which reading that page resets; then `DATA_REQUEST_CURSOR` where there is another page; then repeated groups delimited by `DATA_FRIEND_LOGIN`, as [Get Roster (800)](Capabilities-Messaging.md#get-roster-800) delimits its entries and as [Reply Shape](Capabilities-Messaging.md#reply-shape) places fields that are not an entry's ahead of them:

- a sender: `DATA_FRIEND_LOGIN`, `DATA_FRIEND_IDENTITY` where shown, then its messages, each delimited by `DATA_MESSAGE_GUID` and carrying `DATA_MESSAGE_BODY`, `DATA_MESSAGE_TIMESTAMP` and, where it was signed, the identity draft's `DATA_MESSAGE_SIGNED`;
- then each approval, given or held: `DATA_FRIEND_LOGIN`, `DATA_FRIEND_IDENTITY` where shown, and `DATA_ROSTER_STATE` = `Accepted`, with no messages.

A page is bounded to fit in one transaction.

### Answer Message Request (TBD)

Request/reply. Acts on one sender's request or approval.

**Request fields:** `DATA_FRIEND_LOGIN` (REQUIRED), `DATA_REQUEST_ACTION` (REQUIRED): `1` approve, `2` delete, `3` end an approval.

**Reply fields:** `DATA_REASON_CODE`: `OK`, `RosterFull` for an approval past the cap, or `AccountNotFound` where there is no such request or approval.

### Clear Message Requests (TBD)

Request/reply. Declines friend requests and deletes message requests received within a span.

**Request fields:** `DATA_SPAN_SINCE` (optional), `DATA_SPAN_BEFORE` (optional), each as `DATA_MESSAGE_TIMESTAMP` is encoded. Neither present clears everything.

**Reply fields:** `DATA_REQUEST_COUNT`, the message requests deleted, and `DATA_FRIEND_REQUEST_COUNT`, the friend requests declined.

Every live session of the caller receives Message Requests Changed, and each declined friend request is told to both parties as a [Friend Response (805)](Capabilities-Messaging.md#friend-response-805) rejection is, which `Messaging.MaxFriendRequests` keeps to a bounded burst.

### Message Requests Changed (TBD)

Server-initiated notification, sent only to sessions that have listed requests, whenever their listing would change: a request arriving or growing, an approval given, ended or received, a request deleted or cleared.

**Fields:** `DATA_REQUEST_COUNT`, the senders waiting.

---

## Additions to the Tables

Transactions, from the 829–839 the extension reserves for growth:

| ID | Name | Direction | Description |
|---|---|---|---|
| TBD | Get Message Requests | Client to Server | List requests and approvals |
| TBD | Answer Message Request | Client to Server | Approve, delete, or end an approval |
| TBD | Clear Message Requests | Client to Server | Clear requests within a span |
| TBD | Message Requests Changed | Server to Client | The listing changed |

Data objects. The limit belongs in the limits sub-block `0x0625`–`0x0627`; the rest I suggest placing in a new messaging block, with the identity draft's (see Status):

| ID | Name | Type | Description |
|---|---|---|---|
| TBD (limits) | `DATA_MAX_REQUEST_MESSAGES` | UInt32 | Server's effective `Messaging.RequestMessages`. Login reply only; absent when the server takes no requests |
| TBD | `DATA_TAKES_REQUESTS` | UInt16 | Own preference in Set Presence (808) and Get Message Requests' reply: `0` = take no requests, `1` = take them |
| TBD | `DATA_REQUEST_CURSOR` | Binary | An opaque paging cursor |
| TBD | `DATA_REQUEST_ACTION` | UInt16 | `1` approve, `2` delete, `3` end an approval |
| TBD | `DATA_REQUEST_COUNT` | UInt32 | A count of requests, as each transaction says |
| TBD | `DATA_FRIEND_REQUEST_COUNT` | UInt32 | Friend requests declined by Clear Message Requests |
| TBD | `DATA_SPAN_SINCE` | UInt64 | Start of a span, seconds since the Unix epoch |
| TBD | `DATA_SPAN_BEFORE` | UInt64 | End of a span, seconds since the Unix epoch |

Reason codes:

| Code | Name | Meaning |
|---|---|---|
| TBD (proposed 17) | `Requested` | The message is a request: success, but not delivered as an IM until the recipient acts |

---

## Changes to the Messaging Extension Document

Adopting this draft requires these edits to [Capabilities-Messaging.md](Capabilities-Messaging.md), so that neither document contradicts the other:

1. **Transaction and field tables:** list the transactions and fields above, and narrow the reserved ranges.
2. **Server Limits Advertisement:** add `DATA_MAX_REQUEST_MESSAGES` to the exceptions to "Clients MUST tolerate any of these fields being absent": its absence means the server takes no requests. A server never sends it as `0`.
3. **Reason Codes:** add `Requested`.
4. **IM Send (810):** for a session that has listed requests, its reply may carry `Requested`, `AccountNotFound` for a missing Login or a recipient that has blocked the caller, and `RosterFull` for a reply that would approve past the cap.
5. **Add Friend (802) and Friend Response (805):** where a server bounds friend requests, 802 may answer `RateLimited`, and 805 `RosterFull`.
6. **Server Configuration and `MaxRosterSize`:** list the settings above. `MaxRosterSize` counts approvals, and, where a server bounds friend requests, counts a pending request toward its sender alone.
7. **Persistence:** add requests, approvals and holds to the per-account state, as in [Accounts](#accounts).

---

## Security Considerations

- **Spam control changes.** The extension relies on mutual acceptance, so that only friends can queue messages. Requests let strangers write, so they are held apart from the offline queue, raise nothing a client shows as unread, carry text only, and are bounded per sender, per address, per day and per recipient. A recipient who wants none turns them off.
- **Requests tell the sender nothing.** The answer, and its timing, are the same whether the request was stored, held or dropped, so a sender cannot learn that it was deleted, held, or never seen. A block reads as a missing account, and a missing account reads the same.
- **Taking requests is visible.** Whether an account takes requests, which means a client of it has listed them, is visible to anyone who sends one, as `DATA_DISCOVERABLE` is.
- **An approval is narrower than a friendship.** It reveals no presence, and every send answering `OfflineQueued` keeps an approved stranger from learning when the recipient is signed in, apart from what the recipient's replies and `Read` receipts say.
- **Holds outlast a deletion.** A deleted or declined sender cannot try again for the retention period, so deleting someone is final for that long.
- **Requests can be reported.** A request is text, never sealed, so a server that supports reporting can show a moderator what was sent.

## Client Behavior

No client change is required: a session that never lists requests sees what it sees today. A client that supports this draft, on a server that confirmed bit 6:

1. Lists requests with Get Message Requests once after each login, when the server advertised `DATA_MAX_REQUEST_MESSAGES`, and offers no requests view otherwise. Because listing makes the account take requests, it asks the user first, or sends `DATA_TAKES_REQUESTS` = `0` in its 808 until the user has chosen.
2. Treats `Requested` as a success and shows the message as sent, not delivered.
3. Shows requests apart from conversations, with approve, add friend, delete and block, and a way to clear them in bulk.
4. Uses `DATA_MAX_REQUEST_MESSAGES` to tell the user how many messages a request may hold.

## Implementation Notes

- hxd-ng will implement this draft; its design is in [docs/instant-messaging.md](https://github.com/mishan/hxd-ng/blob/main/docs/instant-messaging.md), including how it keeps a request's answer independent of what became of it.
- A server is easiest to keep honest if it records every request it is sent, stored or not, and counts its bounds from that record, never from the messages that survive.
- Version 2 of the [identity draft](Capabilities-Messaging-Identity.md) proposes friendships that move with a person's identity from server to server. The two drafts are independent; where both are implemented, an offer on a friendship is outside these bounds, and a crossing request that carries one waits for an acceptance.
