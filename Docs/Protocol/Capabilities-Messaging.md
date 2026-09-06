# Instant Messaging Extension

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document describes the instant messaging extension to the Hotline protocol. It adds a presence-based buddy-list messaging layer - friends, presence, one-to-one instant messages with offline store-and-forward, user-to-user file transfer, and call signaling - on top of an otherwise unmodified Hotline server. The extension is optional and capability-gated: a client that does not negotiate it observes no change in behaviour, and no new transaction is ever sent to it.

For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md).

## Table of Contents

- [Background](#background)
- [Architecture](#architecture)
  - [Identity Model](#identity-model)
  - [Server Routing](#server-routing)
  - [Reuse of Existing Subsystems](#reuse-of-existing-subsystems)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
  - [Capability Bits](#capability-bits)
  - [Access Privilege](#access-privilege)
  - [Server Configuration](#server-configuration)
    - [Server Limits Advertisement](#server-limits-advertisement)
- [Identity and Addressing](#identity-and-addressing)
- [Transaction Types](#transaction-types)
- [Data Objects](#data-objects)
  - [Packed `DATA_CALL_MEMBERS`](#packed-data_call_members)
- [Enumerations](#enumerations)
  - [Presence State](#presence-state)
  - [Roster State](#roster-state)
  - [Acknowledgement Type](#acknowledgement-type)
  - [Typing State](#typing-state)
  - [Reason Codes](#reason-codes)
- [Transaction Semantics](#transaction-semantics)
  - [Reply Shape](#reply-shape)
- [Roster and Friend Management](#roster-and-friend-management)
  - [Get Roster (800)](#get-roster-800)
  - [Roster Entry (801)](#roster-entry-801)
    - [An entry group is complete, not incremental](#an-entry-group-is-complete-not-incremental)
  - [Add Friend (802)](#add-friend-802)
  - [Remove Friend (803)](#remove-friend-803)
  - [Friend Request (804)](#friend-request-804)
  - [Friend Response (805)](#friend-response-805)
  - [Block User (806)](#block-user-806)
  - [Unblock User (807)](#unblock-user-807)
  - [Set Friend Nickname (824)](#set-friend-nickname-824)
- [User Discovery](#user-discovery)
  - [Find User (822)](#find-user-822)
  - [User Search (823)](#user-search-823)
- [Presence](#presence)
  - [Set Presence (808)](#set-presence-808)
    - [Clearing the status text](#clearing-the-status-text)
  - [Presence Changed (809)](#presence-changed-809)
- [Instant Messaging](#instant-messaging)
  - [IM Send (810)](#im-send-810)
  - [IM Deliver (811)](#im-deliver-811)
  - [IM Acknowledge (812)](#im-acknowledge-812)
  - [IM Typing (813)](#im-typing-813)
  - [Offline Delivery](#offline-delivery)
- [User-to-User File Transfer](#user-to-user-file-transfer)
  - [File Offer (814)](#file-offer-814)
  - [File Accept (815)](#file-accept-815)
  - [File Decline (816)](#file-decline-816)
  - [File Ready (817)](#file-ready-817)
  - [Relay Path](#relay-path)
    - [The splice is not always a byte copy](#the-splice-is-not-always-a-byte-copy)
    - [Handshake Flags on the Relay](#handshake-flags-on-the-relay)
  - [Direct Path](#direct-path)
- [Call Signaling](#call-signaling)
  - [Call Invite (818)](#call-invite-818)
    - [Call rooms are private to their participants](#call-rooms-are-private-to-their-participants)
  - [Call Accept (819)](#call-accept-819)
  - [Call Decline (820)](#call-decline-820)
  - [Call Cancel (821)](#call-cancel-821)
    - [Who each of these fields names](#who-each-of-these-fields-names)
- [User Profiles](#user-profiles)
  - [Visibility](#visibility)
  - [Display name precedence](#display-name-precedence)
  - [Get User Info (825)](#get-user-info-825)
  - [Set User Info (826)](#set-user-info-826)
  - [Searching profiles](#searching-profiles)
- [Persistence](#persistence)
- [Security Considerations](#security-considerations)
- [Client Behaviour](#client-behaviour)
- [Server Behaviour](#server-behaviour)
- [Implementation Notes](#implementation-notes)

---

## Background

The original Hotline protocol provides synchronous, online-only private messages (`TranSendInstantMsg`, 108) addressed by the ephemeral 16-bit user ID. A message to a user who is offline is dropped, there is no buddy list, no presence, no delivery confirmation, and no concept of a stable per-person identity that survives reconnection.

This extension is more or less a homage to the 1990s instant messengers (ICQ, AIM, MSN): sporting a persistent friend list with mutual-consent authorization, presence, one-to-one messaging with offline store-and-forward, file transfer between friends, and voice/conference call signaling. It does so without altering any existing Hotline behaviour - the new model is layered entirely on top of the existing login, transaction framing, voice room (SFU), and file-transfer subsystems.

---

## Architecture

### Identity Model

The extension distinguishes two pre-existing Hotline identifiers and layers a third concept - a stable bare identity - on top of them. This is analogous to the XMPP bare-JID / full-JID split.

| Layer | Hotline equivalent | Stable across reconnect? | Role |
|---|---|---|---|
| Bare identity | account **Login** | Yes | Roster membership, friend graph, presence aggregation, message and request addressing |
| Resource / endpoint | ephemeral 16-bit **user ID** | No - reassigned each login | Wire-level delivery target; the specific endpoint a call or file transfer connects to |
| Display name | user **name** (nickname) | No - mutable, non-unique | Cosmetic only; never an identity |

The 16-bit user ID is **retained** as the transport address. The server maintains a mapping from each Login to its set of live sessions; addressing a Login resolves to one or more user IDs at delivery time. Adding this indirection does not change the wire framing of any existing transaction.

A Login MAY have multiple concurrent sessions (e.g. a user connected from a vintage and a modern client at once). Text messages and presence fan out to all sessions of a Login; calls and file transfers target a specific session resolved at setup.

### Server Routing

A messaging server maintains an in-memory index:

```
Login (string)  →  set of live sessions (each with a 16-bit user ID)
```

- The index is rebuilt on every login, logout, and reconnect, keyed on the stable Login. A friend's roster entry therefore does not change when that friend's user ID is reassigned.
- **Presence aggregation:** a Login is `online` if its session set is non-empty; `away` only when every session is away; `invisible` is a per-account state that suppresses outbound presence regardless of the set.
- The size of the session set MAY be bounded by configuration (see [Server Configuration](#server-configuration)). A server that bounds it MUST admit the new session and disconnect the **oldest** existing one, rather than refusing the login. A messenger that cannot sign in because a session it can no longer reach - a crashed client, a laptop that was closed - is still counted has no remedy available to it, whereas the person at the newest client is by definition present. Clients MUST therefore treat a disconnect with no preceding error as a possible eviction and MUST NOT retry in a tight loop.

### Reuse of Existing Subsystems

- **Call media** reuses the [voice chat](Capabilities-Voice.md) WebRTC SFU and its transactions (600–606). This extension adds only the *ring* (invite/accept/decline/cancel) semantics that the voice room model lacks.
- **File transfer** reuses the existing Hotline file-transfer port and HTXF handshake. A user-to-user transfer is the same mechanism re-addressed from a server file path to a peer.
- **Text encoding** of all human-readable strings follows the negotiated [text encoding](Capabilities-Text-Encoding.md). Logins SHOULD be restricted to printable ASCII.

---

## Compatibility and Negotiation

### Capability Bits

This extension defines three bits in the `DATA_CAPABILITIES` bitmask (field `0x01F0`). See [DATA_CAPABILITIES](Capabilities.md) for the general negotiation flow.

| Bit | Mask | Name | Description |
|---|---|---|---|
| 6 | `0x0040` | `CAPABILITY_MESSAGING` | Client supports the instant messaging layer: roster, presence, IM, receipts, discovery, and call signaling |
| 7 | `0x0080` | `CAPABILITY_DIRECT_TRANSFER` | Client supports optional peer-to-peer (hole-punched) file transfer; absence means transfers use the server relay |
| 8 | `0x0100` | `CAPABILITY_MESSENGER_SESSION` | The session is a pure instant messenger; the server hides it from the classic user list and suppresses its join/leave announcements |

`CAPABILITY_MESSAGING` is the master switch for the entire extension. A client MUST set bit 6 to participate; the server confirms it in the login reply only when the messaging subsystem is enabled and the account holds the [access privilege](#access-privilege). If the server does not echo bit 6, the client MUST NOT send any transaction defined in this document.

`CAPABILITY_DIRECT_TRANSFER` (bit 7) is meaningful only when bit 6 is also active. It SHOULD be advertised only by clients on platforms capable of NAT traversal; vintage clients SHOULD NOT advertise it and consequently always use the relay path.

`CAPABILITY_MESSENGER_SESSION` (bit 8) declares that this session is a *pure messenger* - it will never join public chat, request the user list, or accept classic PMs. A server that confirms bit 8 (which it MUST do only alongside bit 6) hides the session from the classic Hotline world: the session is excluded from Get User Name List (300) replies and its join/leave/away announcements (Notify Change User 301 / Notify Delete User 302) are suppressed, so regular clients never see a user who cannot be interacted with. The session remains fully reachable through the messaging layer (roster, presence, IM, discovery). Hybrid clients (full Hotline clients that also implement messaging) SHOULD NOT set bit 8.

Voice calls additionally require `CAPABILITY_VOICE` (bit 2). A client that does not advertise bit 2 does not support voice; the messaging layer functions fully without it, and such a client MUST NOT send or be sent call-signaling transactions.

### Access Privilege

Participation is gated per account by a privilege bit in `FieldUserAccess` (110):

| Bit | Name | Description |
|---|---|---|
| 58 | `AccessMessaging` | The account MAY use the instant messaging subsystem (roster, presence, IM, discovery, transfer, calls). |

Bits 0–57 are allocated by the base protocol and prior extensions; bit 58 lies in the free range (58–63) of the legacy 8-byte bitmap, so this extension does **not** depend on the 128-bit [extended privilege](Capabilities-Extended-Priv.md) encoding. The server MUST NOT confirm `CAPABILITY_MESSAGING` for an account that lacks `AccessMessaging`.

An account that lacks `AccessMessaging`, or whose Login is shared by multiple concurrent humans (e.g. a `guest` role account), MUST NOT appear in any roster, MUST NOT be returned by discovery, and MUST NOT send or receive messaging transactions.

#### Reserved logins

**The default `admin` and `guest` logins MUST NOT participate in messaging, and a server MUST refuse them regardless of `AccessMessaging`.** The bit being set on either account is an operator error the server corrects, not an instruction it follows: the server MUST NOT confirm `CAPABILITY_MESSAGING` for such a session, MUST NOT return the login from discovery, and MUST refuse a friend request naming it, exactly as it would for an account without the privilege.

This is a `MUST` rather than operator advice because the privilege bit is the wrong place to express it - it is per-account state an administrator edits, in an interface where nothing distinguishes these two accounts from any other, and the consequence of getting it wrong is not visible until it has already happened.

The two accounts fail for different reasons, and each is sufficient on its own:

- **`guest` is shared.** Messaging identity is the Login, and a Login several people use at once cannot be an identity. A roster entry for `guest` names nobody; presence reports that *somebody* is connected; and an offline message queued for `guest` is delivered to whoever signs in next, which is a stranger. There is no correct behaviour available to a client here, so the only safe answer is that the account never enters the system.

- **`admin` is a published handle on the most privileged account.** `DATA_FRIEND_LOGIN` is deliberately *not* obfuscated - it is how users add each other - so a messaging-enabled `admin` is a confirmed-valid administrative login, published to anyone who can run a directory search, with account security resting entirely on the password. This is the same reasoning as the `SHOULD NOT` under [Identity and Addressing](#identity-and-addressing), raised to a `MUST` for the one login every Hotline server is known to have.

Servers SHOULD apply the same treatment to any other account they know to be shared or administrative. The rule names these two because they are the defaults every deployment starts with, and therefore the two an operator is most likely to enable without considering the consequence.

### Server Configuration

The server operator controls the subsystem via configuration. Using Janus as the reference, the settings are:

| Setting | Type | Default | Description |
|---|---|---|---|
| `Messaging.Enabled` | bool | `false` | Master switch for the messaging subsystem |
| `Messaging.MaxSessionsPerLogin` | int | `0` | Maximum concurrent sessions per Login; `0` = unlimited. Exceeding it evicts the oldest session (see [Server Routing](#server-routing)) |
| `Messaging.MaxRosterSize` | int | `500` | Maximum roster entries per account, counting **every** state - accepted, pending in either direction, and blocked |
| `Messaging.MaxMessageBytes` | int | `4096` | Maximum encoded body length of a single IM |
| `Messaging.OfflineRetentionDays` | int | `30` | Prune delivered, and over-age undelivered, messages after this many days |
| `Messaging.MaxOfflinePerRecipient` | int | `500` | Maximum queued undelivered messages per recipient |
| `Messaging.DirectTransfer` | bool | `false` | Permit the optional peer-to-peer file path |

#### Server Limits Advertisement

When the server confirms `CAPABILITY_MESSAGING` in the login reply it MUST also include the fields below, carrying the caps it will actually enforce:

| Field ID | Name | Setting |
|---|---|---|
| `0x0620` | `DATA_MAX_MESSAGE_BYTES` | `Messaging.MaxMessageBytes` |
| `0x0621` | `DATA_MAX_ROSTER_SIZE` | `Messaging.MaxRosterSize` |
| `0x0622` | `DATA_MAX_OFFLINE_QUEUE` | `Messaging.MaxOfflinePerRecipient` |

These exist because the caps are configurable and every one of them is otherwise discovered by being refused. A client that assumes the defaults will let someone write a message, or add a friend, that the server was always going to reject - and the rejection arrives after the work, when the only remedy is to undo it. The [inline media extension](Capabilities-Inline-Media.md#server-limits-advertisement) advertises its limits for the same reason; this closes the same gap for messaging.

The values are advisory in the sense that the server still enforces them on every request - a client cannot gain anything by ignoring them - but clients SHOULD treat them as authoritative for pre-validation.

- Clients MUST tolerate any of these fields being absent and fall back to the defaults in the table above. A server predating this section sends none of them.
- Servers MUST NOT advertise a value larger than they will accept. Advertising a *tighter* value than the live configuration is permitted, which is how an operator throttles clients ahead of a planned reduction.
- Servers MUST NOT send these fields when `CAPABILITY_MESSAGING` was not confirmed.
- A value of `0` means the corresponding cap is unlimited **only where the setting itself defines `0` that way** - `MaxSessionsPerLogin` is the sole such setting, and it is not advertised. For the three fields above, `0` is not a meaningful value and a client receiving one MUST use the default instead.

Clients SHOULD show the message limit as remaining capacity rather than as an error after the fact, and MUST NOT rely on the count of *characters*: `MaxMessageBytes` bounds the **encoded** body, so the same number of characters costs more in a text encoding that is not single-byte.

---

## Identity and Addressing

- A friend is addressed by **Login**, carried in `DATA_FRIEND_LOGIN` (`0x0600`). This is the public handle: users add friends by exact Login. `DATA_FRIEND_LOGIN` is **not** obfuscated (unlike `FieldUserLogin`, 105, used during authentication).
- Because the Login is a public handle for a messaging-enabled account, account security rests entirely on the password. Operators SHOULD NOT reuse a privileged administrative login as a public messaging handle.
- The 16-bit user ID (`FieldUserID`, 103) appears in transactions only where a specific live endpoint is addressed (call setup, transfer setup).
- All timestamps in this extension are carried in `DATA_MESSAGE_TIMESTAMP` (`0x0607`) as an **8-byte big-endian unsigned integer of seconds since the Unix epoch (1970-01-01 UTC)**. This is distinct from the legacy Hotline 8-byte date structure and is always server-authoritative.

---

## Transaction Types

All transaction IDs are chosen from the free 800-block (`0x0320`+), which does not collide with the base protocol (101–355) or existing extensions (voice 600–606, chat history 700, inline media 750–751, GIF icons 1861–1864).

| ID | Hex | Name | Direction | Description |
|---|---|---|---|---|
| 800 | `0x0320` | Get Roster | Client → Server | Request the full roster snapshot |
| 801 | `0x0321` | Roster Entry | Server → Client | One roster entry (snapshot element or delta) |
| 802 | `0x0322` | Add Friend | Client → Server | Request to add a friend by Login |
| 803 | `0x0323` | Remove Friend | Client → Server | Remove a friend or withdraw a pending request |
| 804 | `0x0324` | Friend Request | Server → Client | Inbound authorization request |
| 805 | `0x0325` | Friend Response | Client → Server | Accept or reject an inbound request |
| 806 | `0x0326` | Block User | Client → Server | Block a Login |
| 807 | `0x0327` | Unblock User | Client → Server | Unblock a Login |
| 808 | `0x0328` | Set Presence | Client → Server | Set own presence state and status text |
| 809 | `0x0329` | Presence Changed | Server → Client | A friend's presence changed |
| 810 | `0x032A` | IM Send | Client → Server | Send a message to a friend |
| 811 | `0x032B` | IM Deliver | Server → Client | Deliver a message to a recipient session |
| 812 | `0x032C` | IM Acknowledge | Bidirectional | Delivery or read receipt |
| 813 | `0x032D` | IM Typing | Bidirectional | Typing indicator |
| 814 | `0x032E` | File Offer | Bidirectional | Offer a file to a friend |
| 815 | `0x032F` | File Accept | Bidirectional | Accept a file offer |
| 816 | `0x0330` | File Decline | Bidirectional | Decline or cancel a file offer |
| 817 | `0x0331` | File Ready | Server → Client (relay ref); Bidirectional (direct candidates) | Relay reference, or direct-path ICE candidate exchange |
| 818 | `0x0332` | Call Invite | Bidirectional | Ring one or more friends into a voice room |
| 819 | `0x0333` | Call Accept | Bidirectional | Accept a ring |
| 820 | `0x0334` | Call Decline | Bidirectional | Decline a ring |
| 821 | `0x0335` | Call Cancel | Bidirectional | Cancel/withdraw a ring before it is answered |
| 822 | `0x0336` | Find User | Client → Server | Resolve an exact Login |
| 823 | `0x0337` | User Search | Client → Server | Directory search over discoverable accounts |
| 824 | `0x0338` | Set Friend Nickname | Client → Server | Set the caller's local alias for a friend |
| 825 | `0x0339` | Get User Info | Client → Server | Read a Login's profile |
| 826 | `0x033A` | Set User Info | Client → Server | Set the caller's own profile |

Transaction IDs 827–839 are reserved for future messaging growth.

---

## Data Objects

All fields are carried using standard Hotline field framing. Field IDs are chosen from the free `0x0600`-block.

| ID (hex) | Dec | Name | Type | Description |
|---|---|---|---|---|
| `0x0600` | 1536 | `DATA_FRIEND_LOGIN` | String | Bare identity (Login) of a roster peer; not obfuscated |
| `0x0601` | 1537 | `DATA_FRIEND_NICKNAME` | String | Caller's local display alias for a friend |
| `0x0602` | 1538 | `DATA_PRESENCE_STATE` | UInt16 | Presence state - see [Presence State](#presence-state) |
| `0x0603` | 1539 | `DATA_PRESENCE_STATUS_TEXT` | String | Custom status text |
| `0x0604` | 1540 | `DATA_ROSTER_STATE` | UInt16 | Relationship state - see [Roster State](#roster-state) |
| `0x0605` | 1541 | `DATA_MESSAGE_GUID` | Binary (16) | Client-generated message identifier (RECOMMENDED: UUIDv4) |
| `0x0606` | 1542 | `DATA_MESSAGE_BODY` | String | Message text (negotiated text encoding) |
| `0x0607` | 1543 | `DATA_MESSAGE_TIMESTAMP` | UInt64 | Seconds since Unix epoch, server-authoritative |
| `0x0608` | 1544 | `DATA_ACK_TYPE` | UInt16 | Receipt type - see [Acknowledgement Type](#acknowledgement-type) |
| `0x0609` | 1545 | `DATA_TYPING_STATE` | UInt16 | Typing state - see [Typing State](#typing-state) |
| `0x060A` | 1546 | `DATA_FILE_TRANSFER_GUID` | Binary (16) | Transfer session identifier |
| `0x060B` | 1547 | `DATA_FILE_RELAY_REF` | Binary (4) | Relay reference presented on the file-transfer port |
| `0x060C` | 1548 | `DATA_DIRECT_CANDIDATE` | String | JSON-encoded ICE candidate for the direct path |
| `0x060D` | 1549 | `DATA_CALL_ID` | Binary (4) | Voice room identifier associated with a ring |
| `0x060E` | 1550 | `DATA_CALL_MEMBERS` | Binary | Packed participant list (call status) |
| `0x060F` | 1551 | `DATA_REASON_CODE` | UInt16 | Outcome/reason - see [Reason Codes](#reason-codes) |
| `0x0610` | 1552 | `DATA_REQUEST_NOTE` | String | Optional note attached to a friend request |
| `0x0611` | 1553 | `DATA_DISCOVERABLE` | UInt16 | Own discovery preference: 0 = unlisted, 1 = directory-searchable |
| `0x0612` | 1554 | `DATA_SEARCH_QUERY` | String | Directory search term |
| `0x0613` | 1555 | `DATA_FRIEND_CAPABILITIES` | UInt16 | A peer's negotiated `DATA_CAPABILITIES` bits, advertised so the UI can gate call/file actions |
| `0x0614` | 1556 | `DATA_PROFILE_NICKNAME` | String | The name the account holder goes by |
| `0x0615` | 1557 | `DATA_PROFILE_FIRST_NAME` | String | Given name |
| `0x0616` | 1558 | `DATA_PROFILE_LAST_NAME` | String | Family name |
| `0x0617` | 1559 | `DATA_PROFILE_EMAIL` | String | E-mail address; matched in full only, never returned by discovery |
| `0x0618` | 1560 | `DATA_PROFILE_GENDER` | UInt16 | 0 = unspecified, 1 = female, 2 = male, 3 = other |
| `0x0619` | 1561 | `DATA_PROFILE_BIRTHDATE` | Binary (4) | `year:2 \| month:1 \| day:1`; any part `0` means "not given" |
| `0x061A` | 1562 | `DATA_PROFILE_COUNTRY` | String | ISO 3166-1 alpha-2 |
| `0x061B` | 1563 | `DATA_PROFILE_POSTCODE` | String | Post/ZIP code |
| `0x061C` | 1564 | `DATA_PROFILE_LANGUAGE` | String | ISO 639-1; **repeated**, at most 3 |
| `0x0620` | 1568 | `DATA_MAX_MESSAGE_BYTES` | UInt32 | Server's effective `Messaging.MaxMessageBytes`. Login reply only |
| `0x0621` | 1569 | `DATA_MAX_ROSTER_SIZE` | UInt32 | Server's effective `Messaging.MaxRosterSize`. Login reply only |
| `0x0622` | 1570 | `DATA_MAX_OFFLINE_QUEUE` | UInt32 | Server's effective `Messaging.MaxOfflinePerRecipient`. Login reply only |

Field IDs `0x061D`–`0x061F` are reserved for future messaging fields, and `0x0623`–`0x0627` for further server limits. The limits are given their own sub-block rather than the next free general IDs because there are only three of those left, and a server's configurable caps are the category most likely to grow.

Existing fields reused by this extension: `FieldUserID` (103), `FieldUserName` (102), the `FieldFile*` family (200–213) for file metadata in offers, and the `FieldVoice*` family (`0x01F5`–`0x01F9`) for SDP/ICE during a call.

### Packed `DATA_CALL_MEMBERS`

```
UInt16   count
repeat count times:
    UInt16   login_length
    bytes    login            (login_length bytes, negotiated encoding)
    UInt16   user_id          (the member's current 16-bit user ID, 0 if not yet joined)
```

---

## Enumerations

### Presence State

`DATA_PRESENCE_STATE` (`0x0602`):

| Value | Name | Meaning |
|---|---|---|
| 0 | `Offline` | Not present. Sent only in [Presence Changed (809)](#presence-changed-809) to report a friend who has disconnected or is `Invisible`. A client MUST NOT set its own presence to `Offline` in [Set Presence (808)](#set-presence-808). |
| 1 | `Online` | Available |
| 2 | `Away` | Idle / temporarily away |
| 3 | `Invisible` | Appears offline to friends; still receives presence and messages |
| 4 | `Busy` | Do-not-disturb; messages still delivered, UI SHOULD signal reluctance |

### Roster State

`DATA_ROSTER_STATE` (`0x0604`):

| Value | Name | Meaning |
|---|---|---|
| 0 | `Removed` | Sentinel used only in [Roster Entry (801)](#roster-entry-801) deltas to signal removal |
| 1 | `PendingOut` | Caller has sent a request awaiting the target's response |
| 2 | `PendingIn` | An inbound request awaits the caller's response |
| 3 | `Accepted` | Mutual friendship; presence and messaging permitted |
| 4 | `Blocked` | Caller has blocked this Login |

### Acknowledgement Type

`DATA_ACK_TYPE` (`0x0608`):

| Value | Name | Meaning |
|---|---|---|
| 1 | `Delivered` | The message reached a live recipient session |
| 2 | `Read` | The recipient displayed the message to the user |

### Typing State

`DATA_TYPING_STATE` (`0x0609`):

| Value | Name | Meaning |
|---|---|---|
| 0 | `Stopped` | Recipient stopped composing |
| 1 | `Started` | Recipient is composing |

### Reason Codes

`DATA_REASON_CODE` (`0x060F`), used across requests, sends, calls, and transfers:

| Code | Name | Meaning |
|---|---|---|
| 0 | `OK` | Success / no error |
| 1 | `AccountNotFound` | No such Login |
| 2 | `NotMessagingEnabled` | Login exists but lacks `AccessMessaging` |
| 3 | `Blocked` | Target has blocked the caller |
| 4 | `AlreadyFriends` | Already in the `Accepted` state |
| 5 | `RequestPending` | A request is already outstanding |
| 6 | `NotFriends` | The action requires an accepted friendship |
| 7 | `OfflineQueued` | Message stored for later delivery (informational success) |
| 8 | `OfflineUndeliverable` | Recipient offline and the action cannot be queued (call, typing, transfer) |
| 9 | `QueueFull` | Recipient's offline queue is at `MaxOfflinePerRecipient` |
| 10 | `RateLimited` | Caller exceeded a per-account rate limit |
| 11 | `NotDiscoverable` | Target is unlisted; not returned by directory search |
| 12 | `RosterFull` | Caller's roster is at `MaxRosterSize`, counting entries in every state - a roster can be full of people the caller has blocked |
| 13 | `MessageTooLong` | Encoded body exceeds `MaxMessageBytes` |

`MessageTooLong` exists because it is the one refusal a client can act on precisely: it knows exactly which message was rejected and can offer to split or trim it, where a generic failure leaves it able only to report that something went wrong. Servers predating this code return a failure with `FieldError` text and no reason code, so clients MUST treat an oversize body as rejected on the error code alone and use the reason code only to improve the message they show.

---

## Transaction Semantics

This extension uses standard Hotline transaction framing (see [Hotline.md](Hotline.md)). Two patterns are used, following the convention established by the voice extension:

- **Request/reply** (800, 802, 803, 805, 806, 807, 808, 810, 812, 822, 823, 824, 825, 826): the client sends with a unique non-zero task ID and the *is-reply* flag unset; the server replies with the same task ID and the *is-reply* flag set. A server MAY leave the reply's *type* field zero - the reference server does - so clients MUST match a reply to its request by task ID and MUST NOT key off the reply's type.
- **Server-initiated notification** (801, 804, 809, 811, and the relayed halves of 814–821): the server sends asynchronously with task ID `0` and the *is-reply* flag unset. The client does not reply at the transaction layer; application-level acknowledgement (where required) is a separate transaction (e.g. 812).

All multi-byte integers are big-endian. Clients MUST ignore unrecognised fields and MUST NOT reject a transaction solely because it carries unknown fields.

### Reply Shape

Every transaction in this document that has a reply follows one shape, and a client needs all three parts of it to render an outcome:

- **Failure** - the transaction header's error code is non-zero, `FieldError` (100) carries text written for the user, and the reply SHOULD carry `DATA_REASON_CODE`. The error code is what decides success or failure; the reason code is what makes the failure actionable. A client MUST branch on the header's error code and MUST NOT infer failure from the presence or absence of a reason code.
- **Failure with no reason code.** Some failures have no code that fits - a malformed request, a missing required field, an unavailable subsystem. Servers SHOULD still send `FieldError` text, and clients MUST handle a failure reply carrying no `DATA_REASON_CODE` by showing that text rather than a generic message of their own.
- **Success** - the error code is zero. The reply SHOULD carry `DATA_REASON_CODE` = `OK`, but **clients MUST NOT require it**: [Get Roster (800)](#get-roster-800), whose reply is entirely entry groups, does not send one, and neither does a reply whose reason is more specific ([IM Send (810)](#im-send-810) answers `OfflineQueued`, [Get User Info (825)](#get-user-info-825) answers `NotFriends`). A non-`OK` reason code on a *successful* reply is information about the outcome, not an error.

Where a reply carries both a reason code and repeated entry groups ([Find User (822)](#find-user-822), [User Search (823)](#user-search-823)), the reason code precedes them. Entry parsing therefore starts at the first `DATA_FRIEND_LOGIN` and ignores anything before it, which is what the delimiting rule in [Get Roster (800)](#get-roster-800) already requires.

---

## Roster and Friend Management

### Get Roster (800)

Request/reply. Sent by the client after login to retrieve its full roster. The reply enumerates entries as repeated [Roster Entry](#roster-entry-801) field groups, or the server MAY stream them as separate 801 notifications following the reply. Any `PendingIn` entries cause the server to also emit the corresponding [Friend Request (804)](#friend-request-804) notifications.

Every entry carries the friend's Login, roster state, and the caller's local nickname if one is set. `DATA_PRESENCE_STATE`, `DATA_PRESENCE_STATUS_TEXT`, `DATA_FRIEND_CAPABILITIES` and `FieldUserName` are sent only for entries in the `Accepted` state - an unanswered request or a blocked Login discloses nothing beyond its own existence, which the caller already knows.

Within an `Accepted` entry the four are not gated alike:

- `DATA_PRESENCE_STATE` and `DATA_FRIEND_CAPABILITIES` describe a *live session*, so both are omitted for a friend who is offline or `Invisible`.
- `DATA_PRESENCE_STATUS_TEXT` and `FieldUserName` are stored against the *account*, so a server MUST send them whenever it holds a value, whether or not that friend is present. They are the two things about a friend that outlive the session.

**A server MUST include the status text it holds for each accepted friend.** [Presence Changed (809)](#presence-changed-809) is not sufficient on its own: it fires when the status *owner* changes something, so it reaches whoever is signed in at that moment and nobody else. A friend who signs in afterwards has already missed it, and the next 809 may never come - a status set once and left alone generates no further traffic. The roster snapshot is the only point at which such a client can learn the current text, so a server that omits it here leaves the status permanently invisible to everyone who was not online when it was set.

**Request fields:** none.

**Reply fields (repeated per entry):** `DATA_FRIEND_LOGIN`, `DATA_FRIEND_NICKNAME` (optional), `DATA_ROSTER_STATE`, `DATA_PRESENCE_STATE` (optional), `DATA_PRESENCE_STATUS_TEXT` (optional), `DATA_FRIEND_CAPABILITIES` (optional), `FieldUserName` (102, optional - the name the friend goes by; see [Display name precedence](#display-name-precedence)).

Because Hotline transactions carry a flat field list, repeated entries are delimited by `DATA_FRIEND_LOGIN`: every field following a `DATA_FRIEND_LOGIN`, up to the next `DATA_FRIEND_LOGIN` (or the end of the transaction), belongs to that entry. `DATA_FRIEND_LOGIN` therefore MUST be the first field of each entry. This governs the **reused** fields too - `FieldUserName` (102) inside an entry is that entry's friend's name. A client that scans the whole transaction for field 102 instead of the current group gives one friend's name to every row in the list. The same delimiting rule governs the repeated results of [User Search (823)](#user-search-823). A server MAY instead deliver the snapshot as a sequence of individual [Roster Entry (801)](#roster-entry-801) notifications following the reply; clients MUST accept either form.

### Roster Entry (801)

Server-initiated notification carrying a single roster delta: an addition, a relationship-state change, a presence change, or a removal. A removal is encoded with `DATA_ROSTER_STATE` = `Removed` (value `0`) and only `DATA_FRIEND_LOGIN`.

**Fields:** `DATA_FRIEND_LOGIN` (REQUIRED), `DATA_ROSTER_STATE` (REQUIRED), `DATA_FRIEND_NICKNAME` (optional), `DATA_PRESENCE_STATE` (optional), `DATA_PRESENCE_STATUS_TEXT` (optional), `DATA_FRIEND_CAPABILITIES` (optional), `FieldUserName` (102, optional).

#### An entry group is complete, not incremental

**A server MUST send every field it currently holds for the entry, and a client MUST replace its stored entry with what arrives rather than merging into it.** An absent optional field means *this entry has no such value*, never *this value is unchanged*.

The name "delta" describes which entry changed, not which fields did.

This is forced by the encoding. A flat field list has no way to mark a field as unchanged, so a client that merges cannot distinguish "unchanged" from "cleared" - and every optional field here is one a user can clear. Clearing an alias with [Set Friend Nickname (824)](#set-friend-nickname-824) emits an entry with no `DATA_FRIEND_NICKNAME`; against a merging client the old alias survives on screen for as long as the session lasts, and there is no subsequent transaction that would ever dislodge it. The same holds for a cleared status text.

The alternative convention - send the field present-but-empty to mean "cleared", absent to mean "unchanged" - is the rule [Set Presence (808)](#clearing-the-status-text) uses in the *client to server* direction, and it is deliberately not extended here. That rule costs a paragraph of explanation per field and is the extension's most-reported source of bugs; applying it to six fields in the direction that every client must parse would multiply the trap rather than contain it. Complete entries need no per-field rule at all.

It follows that a snapshot entry from [Get Roster (800)](#get-roster-800) and a delta for the same relationship are byte-identical, and a conforming client can apply both with one code path. Servers SHOULD build them from one routine for that reason.

The exception is `Removed`, which carries no fields to be complete about: the client deletes the row and nothing else is meaningful. The state-specific rules in [Get Roster (800)](#get-roster-800) apply here too - a non-`Accepted` entry legitimately carries no presence, status, capabilities or `FieldUserName`, so their absence on such an entry is the complete answer rather than an omission.

### Add Friend (802)

Request/reply. The server first validates that the target exists and holds `AccessMessaging`; if not, the reply carries `AccountNotFound` or `NotMessagingEnabled` and nothing is stored. On success the server creates a `PendingOut` entry for the caller and a `PendingIn` entry for the target, persists both, and delivers a [Friend Request (804)](#friend-request-804) to the target (immediately if online, otherwise on the target's next login). A second request while one is pending returns `RequestPending`; a request to an existing friend returns `AlreadyFriends`; a request to a Login that has blocked the caller returns `Blocked` (the server SHOULD make this indistinguishable from `AccountNotFound` to avoid leaking block state - see [Security Considerations](#security-considerations)).

**Request fields:** `DATA_FRIEND_LOGIN` (REQUIRED), `DATA_REQUEST_NOTE` (optional).

**Reply fields:** `DATA_ROSTER_STATE` = `PendingOut` on success, or `DATA_REASON_CODE` on failure.

### Remove Friend (803)

Request/reply. Ends the relationship: withdraws a `PendingOut` request, rejects a `PendingIn` request, or removes an `Accepted` friend.

Removal is **mutual**. The server MUST delete the relationship in both directions and MUST send a [Roster Entry (801)](#roster-entry-801) with `DATA_ROSTER_STATE` = `Removed` to every live session of *both* parties - including the session that made the request.

**Request fields:** `DATA_FRIEND_LOGIN` (REQUIRED).

**Reply fields:** `DATA_REASON_CODE`.

> **Why mutual, and not unilateral.** A unilateral removal - the caller drops their row, the other party keeps theirs and merely stops receiving presence - is the obvious reading of the operation, and it fails in three ways:
>
> - The removed party goes on displaying the remover in their buddy list, frozen at whatever presence was published last, permanently, because no further presence is ever sent.
> - Authorization for [IM Send (810)](#im-send-810) is evaluated against the *sender's* own relationship row. With that row still present, a removed person can keep messaging the person who removed them. Removing someone has to stop them contacting you; that is most of the point of the operation.
> - Nothing ever tells the removed party, so no client can present the state correctly however carefully it is written.

### Friend Request (804)

Server-initiated notification delivering an inbound authorization request. Delivered immediately if the target is online, otherwise on next login.

**Fields:** `DATA_FRIEND_LOGIN` (the requester), `DATA_FRIEND_CAPABILITIES` (optional), `DATA_REQUEST_NOTE` (optional).

### Friend Response (805)

Request/reply. The caller accepts or rejects a `PendingIn` request. Acceptance transitions both sides to `Accepted` and triggers a presence exchange; rejection drops both rows.

The server MUST send a [Roster Entry (801)](#roster-entry-801) reflecting the new state to every live session of *both* parties, including the session that answered. Notifying only the requester leaves the accepter with no way to learn the outcome: a client that waits to be told, rather than updating its own view optimistically, shows nothing at all until its next sign-in.

**Request fields:** `DATA_FRIEND_LOGIN` (REQUIRED), `DATA_ROSTER_STATE` = `Accepted` to accept or absent/`Blocked` to reject.

**Reply fields:** `DATA_REASON_CODE`.

### Block User (806)

Request/reply. Sets the relationship to `Blocked`. A blocked Login cannot send the caller messages, offers, calls, or friend requests, and cannot see the caller's presence; it is omitted from the caller's discovery results. Blocking an existing friend also removes the friendship.

Like [Remove Friend (803)](#remove-friend-803), this changes two rosters, and the server MUST tell both:

- Every live session of the **caller**, including the one that asked, receives a [Roster Entry (801)](#roster-entry-801) with `DATA_ROSTER_STATE` = `Blocked`. The caller's `DATA_FRIEND_NICKNAME` for that Login is **retained** and MUST appear in that entry, as it does in the [Get Roster (800)](#get-roster-800) snapshot: the alias is what makes a blocked row legible when the caller later decides whether to unblock. (Dropping it from the notification while the snapshot keeps it is the kind of disagreement [complete entries](#an-entry-group-is-complete-not-incremental) exist to prevent - the row appears to lose its alias until the next sign-in restores it.)
- Every live session of the **target** receives a [Roster Entry (801)](#roster-entry-801) with `DATA_ROSTER_STATE` = `Removed`, because their reciprocal row is gone.

The target is told that the relationship ended, never that it ended in a block - a removal and a block are indistinguishable to them, which is what keeps [block-state confidentiality](#security-considerations) intact on the one path that would otherwise announce it.

**Request fields:** `DATA_FRIEND_LOGIN` (REQUIRED).

**Reply fields:** `DATA_REASON_CODE`.

### Unblock User (807)

Request/reply. Clears a `Blocked` relationship, leaving no roster entry (the parties are strangers again). The caller's own sessions receive a [Roster Entry (801)](#roster-entry-801) with `DATA_ROSTER_STATE` = `Removed`; the other party is told nothing, having never been told about the block.

Because no row survives, neither does the caller's alias for that Login. Unblocking is not the inverse of blocking in this one respect - it discards what blocking preserved - and a client SHOULD NOT offer it as a reversible toggle.

**Request fields:** `DATA_FRIEND_LOGIN` (REQUIRED).

**Reply fields:** `DATA_REASON_CODE`.

### Set Friend Nickname (824)

Request/reply. Sets the caller's local alias for a friend. The alias is private to the caller and never shown to the friend. The relationship MUST be `Accepted`; against any other state the server refuses with `NotFriends`.

The server MUST send the updated [Roster Entry (801)](#roster-entry-801) to every live session of the caller, including the one that asked. An alias is a per-account setting rather than a per-session one, so a person aliasing a friend on one client expects to see it on their others.

**Request fields:** `DATA_FRIEND_LOGIN` (REQUIRED), `DATA_FRIEND_NICKNAME` (REQUIRED; empty clears the alias).

**Reply fields:** `DATA_REASON_CODE`.

---

## User Discovery

Discovery answers two questions - "does this handle exist" and "who matches this word" - and it answers neither with anything about who is online. That is the whole difference between a directory and a presence service, and it constrains one field in particular.

**Servers MUST NOT send `DATA_FRIEND_CAPABILITIES` to a caller who is not an accepted friend of the subject.** This applies to [Find User (822)](#find-user-822), [User Search (823)](#user-search-823) and [Get User Info (825)](#get-user-info-825) alike. Capabilities are negotiated per session and therefore exist only while the subject is signed in: a server that reports them has reported that fact, and one that omits them has reported the opposite. There is no value the field can carry that does not answer a question discovery declines to answer, which is why the rule is *omit for non-friends* rather than *report something safe*.

Clients MUST NOT read the field's absence as presence information - against a conforming server it is absent for every non-friend, online or not.

### Find User (822)

Request/reply. Resolves an exact Login. Always available regardless of the target's discovery preference, because the caller must already know the exact handle. The reply does **not** expose the target's presence or online state unless the parties are already friends.

**Request fields:** `DATA_FRIEND_LOGIN` (REQUIRED).

**Reply fields on success:** `DATA_REASON_CODE` = `OK`, `DATA_FRIEND_LOGIN`, `FieldUserName` (the target's display name), and `DATA_FRIEND_CAPABILITIES` **only when the caller is already an accepted friend** (see [User Discovery](#user-discovery)). **On failure:** `DATA_REASON_CODE` (`AccountNotFound` / `NotMessagingEnabled`) with a non-zero header error code.

A target that has blocked the caller is reported as `AccountNotFound`, so a caller cannot use this transaction to detect a block.

### User Search (823)

Request/reply. Directory search over accounts whose own `DATA_DISCOVERABLE` preference is `1`. The query matches Login and/or display name. The server bounds the result count and SHOULD rate-limit the transaction to deter enumeration. Accounts that are unlisted, or that have blocked the caller, MUST NOT appear.

**Request fields:** `DATA_SEARCH_QUERY` (REQUIRED).

**Reply fields (repeated per result):** `DATA_FRIEND_LOGIN`, `FieldUserName` (optional), `DATA_FRIEND_CAPABILITIES` (only for a result who is already an accepted friend - see [User Discovery](#user-discovery)), preceded by `DATA_REASON_CODE` = `OK` (see [Reply Shape](#reply-shape)).

The result bound is server-local and is **not** advertised, so a truncated result set is indistinguishable on the wire from a complete one. Clients MUST NOT present results as the complete set of matching accounts, and SHOULD invite the user to narrow the query rather than implying there is nobody else. The reference server caps a search at 50 results.

---

## Presence

### Set Presence (808)

Request/reply. The client sets its own presence state and optional status text, and MAY set its discovery preference. Presence applies to the Login and is aggregated across the Login's sessions. The server persists the status text and discovery preference and restores them on the next login. The new presence is fanned out as [Presence Changed (809)](#presence-changed-809) to all accepted friends (unless `Invisible`).

The fan-out reaches only the friends who are signed in at that moment. Because the status text is persisted against the account rather than the session, it MUST also appear in the [Get Roster (800)](#get-roster-800) snapshot; a friend who signs in later has no other way to learn it.

**The presence *state* is not persisted.** It belongs to the live session, so a Login that signs in again starts at the server's default - `Online`, since a Login with a live session is online (see [Server Routing](#server-routing)) - and the server announces that to the Login's friends as soon as the session registers, before the client has said anything.

Two obligations follow:

- A client MUST send [Set Presence (808)](#set-presence-808) after every login, **including a reconnect**, restating whatever state the user had chosen. Nothing on the server remembers it. A client that treats presence as a one-time announcement leaves a user who dropped while `Away` reported as `Online` for as long as the session lasts.
- A client SHOULD send it **before** [Get Roster (800)](#get-roster-800), because the interval between the session registering and the client's own value arriving is one in which every friend sees the default. Get Roster is the readiness signal and drains the offline queue whenever it arrives; putting the roster reply and that backlog ahead of the correction only widens the window, and it is widest for the users who have been away longest.

**Request fields:** `DATA_PRESENCE_STATE` (REQUIRED), `DATA_PRESENCE_STATUS_TEXT` (optional), `DATA_DISCOVERABLE` (optional).

**Reply fields:** `DATA_REASON_CODE`.

#### Clearing the status text

`DATA_PRESENCE_STATUS_TEXT` is optional, and the two ways of "sending nothing" mean different things:

| Sent | Meaning |
|---|---|
| Field absent | No opinion - the server keeps the stored text unchanged |
| Field present, empty (or only whitespace) | Clear the stored text |

Servers MUST distinguish the two by the field's **presence in the transaction**, not by whether its value is empty, and clients MUST send a present-but-empty field to clear. Conflating them makes a status message impossible to remove: the clear reads as "no opinion", the old text is retained, and it is then republished to the user's friends for as long as the account exists. This is the one place in the extension where an absent field and an empty field are not interchangeable, which is why it is stated rather than left to the general rule.

The simplest conforming client always sends the field, empty when the user has no status. `DATA_DISCOVERABLE` has no equivalent ambiguity - it is a `UInt16` whose absence is the only way to say "unchanged".

### Presence Changed (809)

Server-initiated notification sent to a client when one of its accepted friends changes presence (including going online/offline). A friend who has disconnected or is in the `Invisible` state is reported with `DATA_PRESENCE_STATE` = `Offline` (`0`), so the two are indistinguishable to the recipient. Presence is aggregated across the friend's sessions: the reported state is the most-available session's (online > busy > away), and `Offline` only when no session is visible.

**Fields:** `DATA_FRIEND_LOGIN` (REQUIRED), `DATA_PRESENCE_STATE` (REQUIRED), `DATA_PRESENCE_STATUS_TEXT` (optional), `DATA_FRIEND_CAPABILITIES` (optional), `FieldUserName` (102, optional).

`FieldUserName` carries the name the friend currently goes by, which is why this transaction is also how a [Set User Info (826)](#set-user-info-826) change reaches their friends. A client MUST apply it to its stored roster entry rather than only to a live presence indicator: the name has to survive that friend going offline, or the buddy list falls back to bare account names for everyone who is not signed in.

---

## Instant Messaging

### IM Send (810)

Request/reply. Sends a one-to-one message to an accepted friend. The server validates friendship and block state, enforces `MaxMessageBytes`, stamps `DATA_MESSAGE_TIMESTAMP`, persists the message, and delivers it via [IM Deliver (811)](#im-deliver-811) to every live session of the recipient. If the recipient has no live session, the message is queued (see [Offline Delivery](#offline-delivery)).

**Request fields:** `DATA_FRIEND_LOGIN` (REQUIRED, recipient), `DATA_MESSAGE_GUID` (REQUIRED), `DATA_MESSAGE_BODY` (REQUIRED).

**Reply fields:** `DATA_REASON_CODE` - `OK` when delivered to at least one live session, `OfflineQueued` when stored for later delivery, or an error (`NotFriends`, `Blocked`, `RateLimited`, `QueueFull`).

The recipient is **not** identified by `DATA_MESSAGE_TIMESTAMP`; dedup is by `DATA_MESSAGE_GUID`. Re-sending the same GUID to the same recipient MUST be idempotent.

### IM Deliver (811)

Server-initiated notification delivering a message to a recipient session.

**Fields:** `DATA_MESSAGE_GUID` (REQUIRED), `DATA_FRIEND_LOGIN` (REQUIRED, the sender), `DATA_MESSAGE_BODY` (REQUIRED), `DATA_MESSAGE_TIMESTAMP` (REQUIRED).

### IM Acknowledge (812)

Request/reply from the recipient's client to the server, which forwards the receipt to the sender. A `Delivered` receipt SHOULD be sent automatically on receipt; a `Read` receipt is sent when the user views the message. The server records the receipt and forwards it to the sender's live sessions; if the sender is offline, a `Read` receipt is retained and forwarded on the sender's next login (a `Delivered` receipt MAY be dropped if the sender is offline).

**Request fields:** `DATA_MESSAGE_GUID` (REQUIRED), `DATA_ACK_TYPE` (REQUIRED), `DATA_FRIEND_LOGIN` (REQUIRED, the original sender).

**Reply fields:** `DATA_REASON_CODE`.

### IM Typing (813)

Notification (no reply). Forwarded to the recipient's live sessions only; never stored and never queued. If the recipient is offline the server silently discards it.

**Fields:** `DATA_FRIEND_LOGIN` (REQUIRED, recipient on send / sender on delivery), `DATA_TYPING_STATE` (REQUIRED).

### Offline Delivery

- A message to a recipient with no live session is persisted with no delivery timestamp and the [IM Send (810)](#im-send-810) reply carries `OfflineQueued`.
- On the recipient's next login, after roster sync, the server drains the recipient's undelivered queue oldest-first and pushes each message via [IM Deliver (811)](#im-deliver-811). A message is marked delivered upon the recipient's `Delivered` acknowledgement.
- The queue is bounded by `MaxOfflinePerRecipient`; when full, further sends return `QueueFull`. Delivered messages and over-age messages are pruned per `OfflineRetentionDays`.
- Offline delivery applies to **text messages only**. Calls, typing, and file transfers require both peers online and are never queued (`OfflineUndeliverable`).

---

## User-to-User File Transfer

Both peers MUST be online and in the `Accepted` state.

### File Offer (814)

Request/reply on the sender's side; server-initiated notification on the recipient's side. The sender offers a file by metadata. The server validates friendship/block, allocates a `DATA_FILE_TRANSFER_GUID`, and forwards the offer to the recipient's sessions.

**Sender request fields:** `DATA_FRIEND_LOGIN` (REQUIRED, recipient), `FieldFileName` (201), `FieldFileSize` (207) or [`DATA_FILESIZE64`](Capabilities-Large-File.md) (`0x01F1`), optional `FieldFileTypeString`/`FieldFileCreatorString` (205/206), `DATA_FILE_TRANSFER_GUID` (REQUIRED).

**Recipient notification fields:** the above, with `DATA_FRIEND_LOGIN` set to the sender.

### File Accept (815)

The recipient accepts an offer. Forwarded to the sender. The server then issues [File Ready (817)](#file-ready-817) to both peers.

**Fields:** `DATA_FILE_TRANSFER_GUID` (REQUIRED).

### File Decline (816)

Either peer declines or cancels an offer. Forwarded to the other peer.

**Fields:** `DATA_FILE_TRANSFER_GUID` (REQUIRED), `DATA_REASON_CODE` (optional).

### File Ready (817)

Server-initiated notification instructing both peers to begin the transfer. For the relay path it carries `DATA_FILE_RELAY_REF`; for the direct path it carries the peer's `DATA_DIRECT_CANDIDATE`(s).

**Server → client (relay):** at acceptance the server issues one File Ready to **each** peer. The two peers receive **distinct** relay references that the server maps to the same transfer session; this lets the server tell the uploading side from the downloading side when the two transfer-port connections arrive. (A single shared reference cannot disambiguate direction.)

**Client → server (direct candidates):** in the direct path, File Ready is also used in the client→server direction: a peer sends a File Ready carrying its `DATA_DIRECT_CANDIDATE` and the server forwards it to the other peer as a (server→client) File Ready. This is a notification - no reply.

**Fields:** `DATA_FILE_TRANSFER_GUID` (REQUIRED), and either `DATA_FILE_RELAY_REF` (relay) or one or more `DATA_DIRECT_CANDIDATE` (direct, with `DATA_FRIEND_LOGIN` of the originating peer on the forwarded copy).

### Relay Path

The default path; the only path used when either peer lacks `CAPABILITY_DIRECT_TRANSFER`. It reuses the existing Hotline file-transfer port and HTXF handshake:

1. Both peers open the transfer port and present the `DATA_FILE_RELAY_REF` from [File Ready (817)](#file-ready-817).
2. The server matches the two connections by reference and **splices** them: the sender's payload is forwarded to the recipient.
3. Because both peers are online, the server does not spool to disk - the relay is a live pipe.

The relay reference is single-use and expires on the existing transfer-port handshake timeout. Large-file mode applies if both peers negotiated `CAPABILITY_LARGE_FILES`.

#### The splice is not always a byte copy

Each peer's transfer-port connection is protected the way *its own control session* is protected, and the two need not match. A vintage messenger on a plaintext connection can be sent a file by a modern one on [HOPE ChaCha20-Poly1305](HOPE-ChaCha20-Poly1305.md); each of them is doing the only thing it knows how to do, and neither is aware of what the other negotiated.

Under AEAD this is not merely a difference of transport - the two sides are in **different crypto domains**. The [per-transfer key](HOPE-ChaCha20-Poly1305.md#per-transfer-key) is derived from the control session's keys and the 4-byte transfer reference, and the relay issues each peer a *different* reference for a reason that has nothing to do with encryption (telling the uploader from the downloader). The consequence is that the two peers derive two different keys, and there is no shared key for them to talk under.

A server MUST therefore terminate each side's protection separately: read and authenticate the sender's frames, and re-frame the plaintext under the recipient's own protection. Concretely, per connection:

| Peer's control session | That peer's transfer connection |
|---|---|
| Plaintext | Plaintext |
| HOPE stream cipher (RC4 / Blowfish) | Plaintext - stream mode does not protect transfers |
| HOPE AEAD (ChaCha20-Poly1305) | ChaCha20-Poly1305 under that peer's own per-transfer key, derived from **that peer's** reference |
| TLS | Inside that peer's TLS tunnel |

This is the same work a server already does for an ordinary server-to-client transfer, applied once per side. It is called out because the word *splice* invites the one implementation that cannot work: copying bytes from one socket to the other. That produces a file the recipient cannot decrypt when either side is AEAD, and - because the relay is the one path that never parses the payload - the server has no opportunity to discover it. The failure surfaces at the recipient, as corruption, with nothing in the server's logs.

It also follows that the relay is **not** end-to-end encrypted, and the [Security Considerations](#security-considerations) statement that the server observes relayed content is a property of the design rather than an oversight. A deployment that needs the server not to see the bytes wants the [direct path](#direct-path).

#### Handshake Flags on the Relay

The relay splices payload, not handshakes. Any [handshake extension block](Capabilities-Large-File.md#handshake-flags-and-length) a peer's flags declare is addressed to the *server*, not to the far peer, so the server MUST consume every block the flags call for before splicing, and MUST forward only what follows. A block left on the stream is not a protocol error - it is delivered to the recipient as the opening bytes of their file, which arrives at a plausible size and is wrong from byte zero.

- `HTXF_FLAG_LARGE_FILE` - permitted only when **both** peers negotiated `CAPABILITY_LARGE_FILES`; otherwise the server MUST refuse the transfer. It describes how the *sender* framed the payload, and the recipient must be able to parse that framing. There is no third option: the flag cannot be stripped, because stripping it would not change the bytes the sender is about to write, and it cannot be ignored, because the relay is the one place in the protocol that never parses the payload and so would never discover the disagreement. The recipient would write a mis-parsed file and report success.
- `HTXF_FLAG_SIZE64` - permitted under the same condition, and refused the same way. The large-file extension makes it [valid only alongside `HTXF_FLAG_LARGE_FILE`](Capabilities-Large-File.md#handshake-flags-and-length), so it inherits that flag's authorization rather than carrying one of its own. When permitted, the server consumes the 8-byte length and does not forward it.
- `HTXF_FLAG_RESUME` - **MUST be refused**, unconditionally, however the two peers are capable. A relay is a live pipe between two online peers: nothing is spooled to disk, so there is no partial, no offset the server could have quoted, and nothing to continue. A sender that sets it is about to send a file's tail, and splicing that delivers a truncated file that both peers report as a success. Resuming an interrupted user-to-user transfer means a fresh offer, not a resumed handshake.

This applies to a server that implements no part of the large-file extension as well. A peer may still *set* these flags, and a server that ignores a block instead of consuming it corrupts the transfer.

**When the capability pair is evaluated.** The server MUST decide whether large-file mode is permitted for a relay **when it accepts the transfer** - at [File Accept (815)](#file-accept-815), where it issues the two relay references - and MUST apply that decision to both handshakes when they arrive. Deciding again per connection is not equivalent: the two peers connect at different moments, either may have signed on again in between, and large files may be a per-account privilege that an operator changed. A server that re-evaluates can permit one side and refuse the other, which is the one outcome that yields a half-written file.

A server MUST refuse the flags on a relay reference it does not recognise, rather than treating the absence of a session as an absence of restriction.

**Peers with several sessions.** A Login MAY have more than one live session (see [Offline Delivery](#offline-delivery)), and they need not have negotiated the same capabilities. For this decision a peer counts as capable when **any** of its live sessions negotiated `CAPABILITY_LARGE_FILES`, because the peer chooses which of its sessions answers the [File Ready (817)](#file-ready-817) and only a capable one can act on it. This is deliberately the permissive reading: the handshake is still checked when the transfer connection arrives, so this decides what is *permitted*, not what is *trusted*.

**Recipients that cannot receive.** A client offered a file whose size requires large-file mode, on a session for which the server did not confirm `CAPABILITY_LARGE_FILES`, SHOULD decline the offer at [File Decline (816)](#file-decline-816) rather than accept it and fail at the handshake. It cannot set the flag, and without the flag it cannot parse what the sender will send. Declining tells the sender something actionable while the transfer still costs nothing; failing at the handshake spends a rendezvous to reach the same place. Senders already face the mirror of this rule - a client MUST NOT offer a file it is not authorised to frame.

### Direct Path

Optional; used only when **both** peers advertise `CAPABILITY_DIRECT_TRANSFER` and the server permits it (`Messaging.DirectTransfer`). The server acts solely as a rendezvous broker:

1. After acceptance, each peer sends its `DATA_DIRECT_CANDIDATE`(s); the server forwards them to the other peer via [File Ready (817)](#file-ready-817).
2. The peers attempt a direct connection (hole punching).
3. On any failure (NAT type, timeout), both peers MUST fall back to the [relay path](#relay-path).

---

## Call Signaling

Voice calls require `CAPABILITY_VOICE` (bit 2) on all participating clients, in addition to `CAPABILITY_MESSAGING`. Call media uses the existing [voice chat](Capabilities-Voice.md) SFU and its transactions (600–606); the transactions below add only the ring semantics. Friendship is REQUIRED to ring a peer.

### Call Invite (818)

Request/reply on the caller's side; server-initiated notification (a "ring") on each invitee's side. The caller names one or more friends to invite; **the server allocates the voice room** and returns its `DATA_CALL_ID` in the reply. Each invitee's live sessions all ring, carrying the same ID.

The ID MUST be unpredictable and MUST NOT collide with any room already in use - another call, a private chat, that chat's voice room, or `0` (the public chat). Servers allocate private chat IDs the same way and for the same reason.

A caller MAY include a `DATA_CALL_ID` to invite further friends into a call already under way. The server MUST honour it only when the caller is already a member of that call's room, and MUST otherwise refuse; it MUST NOT treat an unrecognised ID as a request to create a room with that ID.

#### Call rooms are private to their participants

A call's `DATA_CALL_ID` doubles as the voice room ID, and a voice room is addressed by a bare identifier with no membership of its own. Servers MUST therefore enforce the room's guest list themselves:

- A server MUST refuse [Join Voice Room (600)](capabilities-voice.md) for a call's room from any Login that was not the caller or one of the invitees - however privileged, and whether or not it negotiated `CAPABILITY_MESSAGING`. A classic Hotline client with voice support has no concept of this extension and MUST NOT be able to enter a call by naming its ID.
- The guest list MUST outlive the ring. A one-to-one call leaves the ring state the moment it is answered, which is *before* either party joins the room; authorising against ring state alone admits nobody. It is retired when the room empties.
- Inviting further friends into a call already under way MUST add to the guest list rather than replace it.
- Because the *caller* chooses the ID, a server MUST reject a Call Invite naming a voice room that is already occupied, unless the caller is already a member of it. Otherwise a crafted invite naming a private chat's ID - or `0`, the public chat - would walk the invitees into that room.

Conversely, a private chat's voice room MUST be restricted to that chat's members, and only the public chat's room (`0`) is open to any client holding the voice privilege.

**Caller request fields:** one or more `DATA_FRIEND_LOGIN` (REQUIRED - one for a 1:1 call, several for a group/conference call); `DATA_CALL_ID` (optional, and only to add invitees to a call the caller is already in).

**Caller reply fields:** `DATA_CALL_ID` (REQUIRED - the allocated room), or `DATA_REASON_CODE` on failure.

**Invitee notification fields:** `DATA_CALL_ID` (REQUIRED), `DATA_FRIEND_LOGIN` (the caller), `DATA_CALL_MEMBERS` (optional, the full invite set).

### Call Accept (819)

An invitee accepts. The server notifies the caller and the invitee then joins the room via [Join Voice Room (600)](Capabilities-Voice.md). For a Login with multiple ringing sessions, the first to accept wins; the server sends [Call Cancel (821)](#call-cancel-821) to that Login's other sessions.

**Invitee request fields:** `DATA_CALL_ID` (REQUIRED).

**Caller notification fields:** `DATA_CALL_ID` (REQUIRED), `DATA_FRIEND_LOGIN` (REQUIRED - *which* invitee accepted).

### Call Decline (820)

An invitee declines. Forwarded to the caller. In a group call, the call continues for other invitees.

**Invitee request fields:** `DATA_CALL_ID` (REQUIRED), `DATA_REASON_CODE` (optional).

**Caller notification fields:** `DATA_CALL_ID` (REQUIRED), `DATA_FRIEND_LOGIN` (REQUIRED - *which* invitee declined), `DATA_REASON_CODE` (optional, forwarded unchanged).

### Call Cancel (821)

Sent by the caller to withdraw a ring before it is answered, or by the server to silence a Login's other sessions once one has accepted, or to signal the caller hung up. Only the caller may cancel a call; a cancel from anyone else MUST be ignored.

**Caller request fields:** `DATA_CALL_ID` (REQUIRED).

**Invitee notification fields:** `DATA_CALL_ID` (REQUIRED), `DATA_FRIEND_LOGIN` (the caller). The server's cancel to a Login's *own* other sessions after one of them accepted carries `DATA_CALL_ID` alone - there is no other party to name, and the absence of `DATA_FRIEND_LOGIN` is how a client tells "the caller hung up" from "you answered this somewhere else".

#### Who each of these fields names

`DATA_FRIEND_LOGIN` in 819, 820 and 821 is the **other party**, never the recipient of the notification. A one-to-one call works without reading it at all, which is why it was possible to omit it from this document for as long as it was; a group call does not, because "somebody declined" is not a thing a client can display. Servers MUST send it and clients MUST tolerate its absence from an older server, falling back to the one-to-one assumption.

---

## User Profiles

An account MAY publish a small profile: a nickname, a given and family name, an e-mail address, gender, date of birth, country and post code, and the languages they speak. **Every field is optional.** The Login is not part of the profile - it is the account's permanent identity and can never be changed.

Three details are worth stating rather than leaving to each implementation:

- **The birth date's parts are independently optional.** `DATA_PROFILE_BIRTHDATE` carries year, month and day, and any of them may be `0` for "not given". Somebody willing to publish a day and month but not a year is an ordinary thing to want, and a single opaque date would make it all or nothing. Servers MUST reject an impossible month or day rather than store it.
- **Languages are a repeated field, capped at three.** The subject of these transactions is one account, so a repeat is unambiguous - there is no entry list for it to be confused with. Servers MUST truncate rather than reject a longer list.
- **Gender is self-selected and deliberately short.** It exists so people can say something about themselves, not so a server can categorise them. An unrecognised value MUST be stored as unspecified rather than rejected, so a newer client cannot be refused by an older server.
- **Servers bound each profile string and truncate rather than refuse.** The bound is server-local and not advertised; the reference server uses 128 bytes per field. Truncating keeps [Set User Info (826)](#set-user-info-826) from being an all-or-nothing operation over one long entry, but it does mean a client cannot assume what it wrote is what it will read back, and SHOULD refresh its own profile from [Get User Info (825)](#get-user-info-825) after saving rather than trusting its local copy.

### Visibility

**A profile is readable by accepted friends and by nobody else.** To anyone else the server returns the same public card that discovery already returns - `DATA_FRIEND_LOGIN` and `FieldUserName` - plus `DATA_REASON_CODE` = `NotFriends` so the client can explain the absence rather than render an empty panel. `DATA_FRIEND_CAPABILITIES` is **not** part of the non-friend card, for the reason given under [User Discovery](#user-discovery): it exists only for a live session, so sending it to a stranger reports that the subject is signed in.

This is deliberately a single rule rather than per-field visibility. A directory that publishes more than its users expect is the failure mode to avoid, and one rule is a rule every client implements the same way.

### Display name precedence

A client SHOULD show, in order: the viewer's own alias for that person (`DATA_FRIEND_NICKNAME`, private to the viewer), then the name the person publishes, then the bare Login. The Login SHOULD NOT be appended to either of the first two - it is an identifier, not part of anybody's name.

Servers SHOULD send the published name as `FieldUserName` in [Roster Entry (801)](#roster-entry-801) and [Presence Changed (809)](#presence-changed-809), falling back to the display name of a live session. Because the stored name outlives the session, this is what allows a client to show something better than an account name for a friend who is offline.

### Get User Info (825)

Request/reply. Reads a Login's profile.

**Request fields:** `DATA_FRIEND_LOGIN` (REQUIRED).

**Reply fields:** `DATA_FRIEND_LOGIN` and `FieldUserName` always; plus `DATA_FRIEND_CAPABILITIES` and the `DATA_PROFILE_*` fields when the caller is an accepted friend (or is the subject), or `DATA_REASON_CODE` = `NotFriends` when not. Servers MUST rate-limit this alongside the other discovery transactions.

### Set User Info (826)

Request/reply. Replaces the caller's own profile; an absent or empty field clears that entry. The subject is always the authenticated Login of the session - there is no field naming whose profile to write, so one account can never write another's.

**Request fields:** any of `DATA_PROFILE_NICKNAME`, `DATA_PROFILE_FIRST_NAME`, `DATA_PROFILE_LAST_NAME`, `DATA_PROFILE_EMAIL`, `DATA_PROFILE_GENDER`, `DATA_PROFILE_BIRTHDATE`, `DATA_PROFILE_COUNTRY`, `DATA_PROFILE_POSTCODE`, and up to three `DATA_PROFILE_LANGUAGE`.

**Reply fields:** `DATA_REASON_CODE`.

Because the published name appears in friends' rosters, a server MUST announce a change to the caller's accepted friends - a [Presence Changed (809)](#presence-changed-809) carrying the new `FieldUserName` is sufficient.

### Searching profiles

[User Search (823)](#user-search-823) matches a discoverable account's nickname, given name and family name on substring, exactly as it already matches Login and display name.

Gender, birth date, country, post code and languages are **not** searchable. They are things a friend may read, not selectors for finding strangers: a directory that can be filtered by age and location is a directory built for a purpose this one does not have.

**E-mail matches only in full.** A substring match would let anyone walk the directory and confirm addresses a fragment at a time - searching `@example.com` would enumerate an entire organisation. Requiring the whole address means the searcher already had it, which is the same reasoning that makes [Find User (822)](#find-user-822) exact-match.

Results carry no profile detail whatsoever: finding someone is not the same as being allowed to read about them.

---

## Persistence

This section is informative; the on-disk representation is a server implementation detail. A conforming server MUST persist the friend graph and undelivered messages across restarts. Presence, typing, calls, and in-flight transfers are live-only.

A server SHOULD also persist, per account: the discovery preference (`DATA_DISCOVERABLE`), the published [profile](#user-profiles), and the last self-set status text, restoring them on login.

Three account-lifecycle events reach into the friend graph, and each MUST leave it consistent:

- **Deletion.** Remove all roster rows referencing the Login (in either direction), purge its undelivered messages, and notify online friends with a removal [Roster Entry (801)](#roster-entry-801).
- **Rename.** Rewrite all references atomically and notify affected online friends - a removal for the stale Login followed by a fresh entry for the new one, since the Login is the roster key and a client cannot rename a row it identifies by that key.
- **Revocation of `AccessMessaging`.** Treat as a deletion of the messaging identity: the account can no longer participate, so leaving its rows in other people's rosters would leave friends addressing a Login the server will refuse every transaction for.

---

## Security Considerations

- **Authorization is server-enforced.** The server MUST verify friendship and block state on every message, file offer, call, and friend request. The client is never trusted to enforce these.
- **Block-state confidentiality.** To avoid revealing that a target has blocked the caller, a server SHOULD return `AccountNotFound` rather than `Blocked` for friend requests and discovery against a blocking target, so a blocked caller cannot distinguish a block from a non-existent account.
- **Enumeration resistance.** [Find User (822)](#find-user-822) and [User Search (823)](#user-search-823) MUST be rate-limited per account. Directory search MUST return only opted-in accounts. Exact-handle resolution leaks only existence and capabilities, never presence, for non-friends.
- **Spam control.** Because messaging requires mutual acceptance, only accepted friends can queue offline messages. Offline queues are bounded (`MaxOfflinePerRecipient`), and per-account send rate limits SHOULD reuse the server's existing flood-control mechanism.
- **Public handle.** A messaging-enabled account's Login is a public handle; account security rests entirely on the password. Operators SHOULD NOT reuse a privileged administrative login as a messaging handle.
- **Transport encryption.** This extension defines no payload encryption of its own. Over the relay path the server observes message and file content, identical to the trust model of the base Hotline protocol. Deployments requiring confidentiality on the wire SHOULD use the [HOPE](HOPE-Secure-Login.md) encrypted transport. The optional direct file path MAY provide end-to-end confidentiality via the WebRTC data channel's DTLS, but this is never mandatory (vintage clients cannot rely on it).
- **Identifier spoofing.** The server MUST derive the authenticated Login from the session, never from a client-supplied `DATA_FRIEND_LOGIN`, when attributing the *sender* of any transaction.

---

## Client Behaviour

A conforming messaging client:

1. Advertises `CAPABILITY_MESSAGING` (and optionally `CAPABILITY_VOICE`, `CAPABILITY_DIRECT_TRANSFER`, `CAPABILITY_TEXT_ENCODING`, `CAPABILITY_LARGE_FILES`) during Login (107). If the server does not confirm `CAPABILITY_MESSAGING`, the client MUST disable all messaging UI and MUST NOT send messaging transactions.
2. Restates its presence with [Set Presence (808)](#set-presence-808) after every login, reconnects included, then issues [Get Roster (800)](#get-roster-800) and handles streamed [Roster Entry (801)](#roster-entry-801) and pending [Friend Request (804)](#friend-request-804) notifications.
3. Gates call and direct-transfer UI on each friend's `DATA_FRIEND_CAPABILITIES`: it MUST NOT offer a voice call to a friend lacking `CAPABILITY_VOICE`, nor a direct transfer to one lacking `CAPABILITY_DIRECT_TRANSFER`.
4. Generates a unique `DATA_MESSAGE_GUID` per message and retries idempotently using the same GUID.
5. Sends `Delivered` receipts automatically and `Read` receipts on display.
6. Ignores unrecognised fields and transaction types.
7. Decides success from the transaction header's error code, not from `DATA_REASON_CODE`, and shows `FieldError` (100) text when a failure carries no reason code (see [Reply Shape](#reply-shape)).
8. Parses repeated entries by group, attributing each `FieldUserName` (102) to the `DATA_FRIEND_LOGIN` that opened its group, and stores the name and status text so offline friends still have both.
9. Replaces a stored roster entry with the group that arrives in [Roster Entry (801)](#roster-entry-801) rather than merging into it, so a cleared alias or status clears (see [An entry group is complete, not incremental](#an-entry-group-is-complete-not-incremental)).
10. Sends `DATA_PRESENCE_STATUS_TEXT` present-but-empty to clear a status, never omitted (see [Clearing the status text](#clearing-the-status-text)).

## Server Behaviour

A conforming messaging server:

1. Confirms `CAPABILITY_MESSAGING` only when the subsystem is enabled and the account holds `AccessMessaging`.
2. Maintains the Login→sessions index and aggregates presence across sessions.
3. Persists the friend graph and undelivered messages; drains a recipient's offline queue on login.
4. Enforces friendship, block state, roster/message/queue limits, and rate limits.
5. Stamps `DATA_MESSAGE_TIMESTAMP` authoritatively and dedups by `(GUID, recipient)`.
6. Maintains account-lifecycle integrity on delete/rename/revoke.
7. Never sends a messaging transaction to a client that has not negotiated `CAPABILITY_MESSAGING`.
8. Answers failures with a non-zero header error code plus `FieldError` (100) text, and a `DATA_REASON_CODE` wherever one applies (see [Reply Shape](#reply-shape)).
9. Sends `FieldUserName` (102) and `DATA_PRESENCE_STATUS_TEXT` with roster entries as well as presence changes, so both reach a client that signed in after they were last set. For the name it prefers the published profile over a live session's, so the value outlives the session.
10. Withholds `DATA_FRIEND_CAPABILITIES` from a caller who is not an accepted friend of the subject, on every discovery transaction (see [User Discovery](#user-discovery)).
11. Terminates transfer-port protection per peer on a relay rather than copying bytes between the two connections (see [The splice is not always a byte copy](#the-splice-is-not-always-a-byte-copy)).

## Implementation Notes

- **Reference allocations.** In the Janus reference server the capability bits, access bit, transactions, and fields are allocated as in this document.
- **Roster removal encoding.** [Roster Entry (801)](#roster-entry-801) signals a removal with the reserved `DATA_ROSTER_STATE` value `0` (`Removed`); all other values are live relationship states. A removal is therefore a state, not a separate transaction, and a client applying deltas needs no special case for it beyond deleting the row.
- **Call media.** Once a [Call Accept (819)](#call-accept-819) completes, all subsequent media negotiation and transport follow the [voice chat extension](Capabilities-Voice.md) unchanged; this extension does not alter SDP/ICE/SFU behaviour.
- **Date encoding.** `DATA_MESSAGE_TIMESTAMP` deliberately uses a plain Unix-epoch UInt64 rather than the legacy Hotline 8-byte date structure, to avoid the 1904-epoch ambiguity described in [DATA_CAPABILITIES](Capabilities.md#date-format-selection). The field is Unix seconds unconditionally: it is a distinct encoding rather than a negotiated reading of the Hotline date, so `CAPABILITY_MODERN_DATES` does not apply to it and a client need not advertise that bit to decode it.
- **Offline drain trigger.** The Janus reference server drains a recipient's offline queue and forwards pending read receipts in response to [Get Roster (800)](#get-roster-800) - the "client is ready" signal a conforming client always sends after login. A client that never requests its roster will not receive its backlog.
- **Per-peer relay references.** The reference server issues a distinct `DATA_FILE_RELAY_REF` to each peer (mapped to one session) so the transfer-port splice can tell the uploader from the downloader; see [File Ready (817)](#file-ready-817). Because the AEAD per-transfer key is derived from the reference, the two peers necessarily hold different keys, and the reference server bridges the two domains rather than forwarding frames.
- **Session eviction.** The reference server enforces `MaxSessionsPerLogin` by evicting the oldest session when a new one would exceed the cap, logging the eviction against the displaced session.
- **Rate limiting.** Per-account token-bucket limits are applied to discovery ([Find User](#find-user-822) / [User Search](#user-search-823)) and to [IM Send (810)](#im-send-810), returning `RateLimited` when exceeded.
- **Operator surface.** The reference server exposes admin/metrics over its REST API - messaging counters (IM volume, offline-queue depth) and per-account roster inspection for moderation - and routes the messaging transactions through its Lua plugin hooks (`pre_*` to reject, e.g. anti-spam on IM Send; `on_*` for auto-reply and logging). These are server-local facilities and not part of the wire protocol.
