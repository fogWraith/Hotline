# Server Linking Extension

> Last updated: October 4, 2026

> **Status:** Accepted, Implemented by the reference server, Janus, from 2.0.18.

> **Developer Note:** This is only the beginning, see [Future Work](#future-work).

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document describes linking independently operated Hotline servers into a **network**, so that their users share one community: every server's users appear in every other server's user list, talk in one public chat, and can send each other private messages. The servers keep their own accounts, files, news and administration. The link joins people, not servers.

**A linked server connects to its peer as a special client.** It dials the peer's ordinary Hotline port, logs in with an account the peer's operator created for it, and declares itself a link with a capability bit. Everything after that rides that session. There is no new listener, no new port and no new framing, and a link works when only one of the two servers can accept inbound connections.

**Links relay.** A server passes on what it learns from one link to its other links, so a chain or a hub reaches everyone: if B links to A and A links to C, the users of all three servers appear on all three. Links form a tree; a link that would close a loop is refused.

**Clients need no changes.** A remote user is presented to local clients as an ordinary user with an ordinary 16-bit user ID - a *ghost* - and every Hotline client since 1.2.3 can see, read and message ghosts with the transactions it already implements.

For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md).

---

## Table of Contents

- [Background](#background)
- [Terminology](#terminology)
- [Architecture](#architecture)
  - [The Link Session](#the-link-session)
  - [Ghosts](#ghosts)
  - [Relaying](#relaying)
  - [User IDs on a Link](#user-ids-on-a-link)
  - [Topology: a Tree](#topology-a-tree)
- [Trust Model](#trust-model)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
  - [Capability Bit](#capability-bit)
  - [Authorizing a Link](#authorizing-a-link)
  - [Protecting the Link](#protecting-the-link)
  - [Link Session Restrictions](#link-session-restrictions)
- [Transaction Types](#transaction-types)
- [Data Objects](#data-objects)
- [Enumerations](#enumerations)
  - [Link Features](#link-features)
  - [Link Reason Codes](#link-reason-codes)
- [Transaction Semantics](#transaction-semantics)
- [Server Identity](#server-identity)
  - [The Server Group](#the-server-group)
  - [Server ID](#server-id)
  - [Tag](#tag)
- [Link Lifecycle](#link-lifecycle)
  - [Link Hello (900)](#link-hello-900)
  - [Link Ping (910)](#link-ping-910)
  - [Link Close (911)](#link-close-911)
  - [Interruption and Resynchronisation](#interruption-and-resynchronisation)
  - [One Link per Peer](#one-link-per-peer)
- [Network Topology](#network-topology)
  - [Link Servers (912)](#link-servers-912)
  - [Link Server Update (913)](#link-server-update-913)
  - [Link Server Gone (914)](#link-server-gone-914)
  - [Loop Prevention](#loop-prevention)
  - [Routing](#routing)
- [User Sharing](#user-sharing)
  - [Who Is Exported](#who-is-exported)
  - [The User Group](#the-user-group)
  - [Exclusion](#exclusion)
  - [Link Snapshot (901)](#link-snapshot-901)
  - [Link User Update (902)](#link-user-update-902)
  - [Link User Gone (903)](#link-user-gone-903)
- [Presenting Ghosts](#presenting-ghosts)
  - [Display Name](#display-name)
  - [Color](#color)
  - [User Flags](#user-flags)
  - [Transactions Naming a Ghost](#transactions-naming-a-ghost)
  - [Ghosts and Other Extensions](#ghosts-and-other-extensions)
- [Public Chat](#public-chat)
  - [Link Chat (904)](#link-chat-904)
- [Private Messages](#private-messages)
  - [Link Private Message (905)](#link-private-message-905)
- [User Info](#user-info)
  - [Link User Info (906)](#link-user-info-906)
- [Moderation](#moderation)
  - [Link Kick (907)](#link-kick-907)
  - [Link Ban (908)](#link-ban-908)
  - [Link Unban (909)](#link-unban-909)
- [Limits](#limits)
- [Security Considerations](#security-considerations)
- [Client Behaviour](#client-behaviour)
- [Server Behaviour](#server-behaviour)
- [Future Work](#future-work)
- [Implementation Notes](#implementation-notes)

---

## Background

Hotline servers have always been islands. Each has its own regulars, who come for its files, its news and the people who have been there for years, and who rarely move. A user who wants to talk to the people on three servers connects to all three, in three windows or three clients, and keeps up with three chats. The three communities never meet.

Operators who wanted a larger community have had two options: merge onto one machine under one operator, which usually nobody's regulars want, or bridge chat to an external network such as IRC, where remote people appear only as relayed lines of text.

This extension links servers instead. Each operator keeps their server, their accounts and their rules. What crosses a link is the *people*: their presence in the user list, their public chat, their private messages. Nothing that belongs to the server itself crosses. A user stays on their own server and finds the whole network's users there.

It is deliberately not account federation. A user signs in to their own server with their own account, exactly as today, and is visible on other servers as a guest of those servers. No credential, password material or privilege ever crosses a link.

---

## Terminology

| Term | Meaning |
|---|---|
| **Link** | An authenticated session between two servers, carrying the transactions in this document |
| **Peer** | The server at the other end of a link |
| **Network** | A set of servers connected by links, directly or through other servers |
| **Dialer** / **Acceptor** | The server that opened the link's connection and logged in, and the server that authenticated it. Only meaningful during establishment |
| **Home server** | The server a user is actually connected to |
| **Local user** | A user whose home server is this server |
| **Ghost** | This server's representation of another server's user: an entry in the user list with a local user ID and no connection of its own |
| **Export** | To announce a user to a peer, so that the peer creates a ghost for them |
| **Relay** | To export, over one link, users and servers learned over another |
| **Server ID** | A server's permanent, network-unique identifier |
| **Tag** | A server's short, human-readable, network-unique name (e.g. `hl2`), used in ghost display names |
| **Epoch** | A random value a server chooses each time it starts, which tells its peers whether the user IDs it sent are still the same ones |

---

## Architecture

### The Link Session

```
   S2 (dialer)                                     S1 (acceptor)
       │  TCP to S1's ordinary Hotline port                │
       │──────────────────────────────────────────────────▶│
       │  Login (107) as account "link-hl2",               │
       │  HOPE with ChaCha20-Poly1305,                     │
       │  DATA_CAPABILITIES = TEXT_ENCODING | SERVER_LINK  │
       │◀──────────────────── login reply, bit 11 confirmed│
       │                                                   │
       │◀──────────── Link Hello (900) ───────────────────▶│  both directions
       │◀──────────── Link Servers (912) ─────────────────▶│  both directions
       │◀──────────── Link Snapshot (901) ────────────────▶│  both directions
       │                                                   │
       │◀═════ symmetric from here: users, chat, PMs ═════▶│
```

Once the login reply confirms the capability, **the link is symmetric**. Which side dialed has no further meaning.

### Ghosts

For every user a peer exports, the receiving server creates a ghost: an entry in its user list with a user ID drawn from the same allocator as local users, a name, an icon and flags, and no socket. Local clients learn about ghosts through Get User Name List (300), Notify Change User (301) and Notify Delete User (302), exactly as they learn about local users.

What a local client does *to* a ghost is translated by its server into a link transaction - a chat line, a private message, a user-info request - and sent toward the ghost's home server, which does the corresponding thing to the real user. What a ghost "does" arrives the other way: a link transaction, which the receiving server presents to its clients as though the ghost had done it.

A ghost holds **no privileges**. It never authenticates to the receiving server and never sends a transaction through the receiving server's ordinary handlers. Its only means of acting is a link transaction, and every such transaction is validated on arrival.

### Relaying

A server exports to each peer not only its own local users but also **the ghosts it learned from its other links**, and forwards the chat lines, messages and moderation requests that go with them. Everything a server learns from a link is passed on to all of its other links, and never back to the link it came from.

```
   B (6 users) ─────── A (11 users) ─────── C (19 users)

   B shows:  6 local + 11 from A + 19 from C (relayed by A)  = 36
   A shows: 11 local +  6 from B + 19 from C                 = 36
   C shows: 19 local + 11 from A +  6 from B (relayed by A)  = 36
```

Relaying across a server is consensual: it happens only when **both** links involved negotiated the [transit feature](#link-features). An operator who wants to link with one server without being joined to that server's other links can switch transit off on their link.

### User IDs on a Link

**A user ID on a link is always the ID used by the server that exported that user over that link.** A server exports its local users under their local IDs and relayed users under its own ghost IDs for them. A peer addressing one of those users uses the same ID back.

Each server therefore keeps, per link, a table from *peer ID* to *ghost ID*, and a transaction relayed across several servers has its user IDs translated at each hop. No server needs to know the IDs any other server uses, and every ID on the wire is an ordinary 16-bit Hotline user ID. An ID is meaningful only together with the exporting server's [epoch](#interruption-and-resynchronisation).

Users are additionally attributed to their **home server** by [server ID](#server-id), which every hop passes along unchanged. The home server is what a client is shown, what moderation is routed to, and what [exclusion](#exclusion) is expressed in.

### Topology: a Tree

**The links of a network MUST form a tree**: between any two servers there is exactly one path. Every server's view of the network is then unambiguous - each other server, and each of its users, is reached through exactly one link - so relayed state never arrives twice and needs no deduplication.

A link that would close a loop is refused when it is established (see [Loop Prevention](#loop-prevention)). Redundant links that stand by to heal a split are [future work](#future-work).

The cost of a tree is that a broken link splits the network: the servers on each side lose sight of the other side until it is restored. The [grace period](#interruption-and-resynchronisation) hides brief interruptions.

---

## Trust Model

**Linking is an agreement between operators, and it is a statement of mutual trust.** With relaying, that trust extends across the network. Operators should understand this before they link:

- **Joining a network means trusting every server in it, through the servers in between.** When A relays B's users to C, C believes what A says about B's users, without C and B ever having made an agreement. Each operator vouches for the peers they link to. The [transit feature](#link-features) is how an operator declines to vouch onward.
- **A server speaks for its own users and relays for others.** A server is trusted to report its own users accurately and to relay faithfully what it learned. It cannot speak for users it never exported, and it cannot address users it was never shown.
- **Moderators are trusted network-wide.** Anyone with the disconnect privilege on any server is a moderator of the whole network. A [kick](#link-kick-907) removes a user from the moderator's own server; a [ban](#link-ban-908) removes them from the network, **including their home server**, which carries it out. A conforming server MUST honour these requests for its own users and MUST relay them faithfully. An operator who would not want another server's moderators banning their regulars should not join that network.
- **Privilege never crosses.** Administrative status, account privileges and access bits are local policy statements and never cross a link in either direction. An administrator on A is an ordinary user everywhere else.
- **Every server on a path sees what crosses it.** Public chat and private messages between users on different servers are readable by their home servers' operators and by every server that relays them. None of it is end-to-end encrypted.
- **Unlinking is unilateral.** Any operator can close any of their links at any time, with no cooperation from anyone.

---

## Compatibility and Negotiation

### Capability Bit

This extension defines one bit in the `DATA_CAPABILITIES` bitmask (field `0x01F0`):

| Bit | Mask | Name | Description |
|---|---|---|---|
| 11 | `0x0800` | `CAPABILITY_SERVER_LINK` | The session is a server link, not a user |

A dialer sets bit 11 in its Login (107). It MUST also set `CAPABILITY_TEXT_ENCODING` (bit 1), and an acceptor MUST NOT confirm bit 11 unless it also confirms bit 1: **all strings on a link are UTF-8**, and each server converts to and from the encodings its own clients negotiated.

A dialer that does not see bit 11 confirmed in the login reply MUST disconnect without sending any further transaction. It SHOULD report the condition to its operator as a configuration error rather than retrying at its ordinary reconnect rate.

A client that is not a server MUST NOT set bit 11.

### Authorizing a Link

**An acceptor MUST confirm bit 11 only when the authenticated account is configured, on the acceptor, as the link account for a known peer.** How that configuration is expressed is an implementation matter.

Authorization is deliberately not an access privilege bit. A privilege bit is per-account state edited in an account editor alongside forty others. Nothing there distinguishes the bit that turns an account into a server, with the power to place a network's worth of users in the user list, from the bit that lets it download files. The reasoning matches the [reserved logins](Capabilities-Messaging.md#reserved-logins) rule of the messaging extension: the privilege bitmap is the wrong place to record a decision whose consequences are this large.

The link account SHOULD hold no ordinary privileges. Under [Link Session Restrictions](#link-session-restrictions) it can use none of them, but an account with none is harmless if it is ever used by mistake as an ordinary login.

Authorization is evaluated continuously, not only at login:

- If the link account is deleted, or disabled on a server that has such a state, or its configuration entry is removed, while the link is up, the acceptor MUST send [Link Close](#link-close-911) with `Unlinked` and end the session.
- **Ordinary connection controls still apply to a link login.** An address ban covering the dialer refuses it like any other connection, so an operator's existing ban tools can also stop a peer. A server MAY exempt configured link accounts from per-address connection limits and from automatic connect-rate bans, since a dialer reconnecting with backoff is not a flood.

  Such limits are usually applied when a connection arrives, before anything shows that it is a link. A dialer's identification is not authenticated, so it cannot be trusted at that point. One way to grant the exemption anyway is to trust an address *after* a link has authenticated from it: from then on, for a bounded time renewed while the link stays up, connections from that address skip the automatic limits, while bans set by an operator still apply. This matters because a dialer's address is often shared, behind one home router, with its operator's own clients, whose reconnections after a restart can otherwise get the address banned for the link as well. The reference server trusts such an address for seven days after the last link login, and withdraws the trust as soon as the peer is removed from its configuration.

### Protecting the Link

A link carries a password that lets its holder place users in the peer's user list. **An acceptor MUST NOT confirm bit 11 on an unprotected session**, and a dialer MUST NOT send link traffic over one. The session MUST be protected in one of two ways:

| Protection | Requirements |
|---|---|
| **HOPE with AEAD** (RECOMMENDED) | [HOPE secure login](HOPE-Secure-Login.md) with the [ChaCha20-Poly1305 transport](HOPE-ChaCha20-Poly1305.md). The dialer MUST abort if the acceptor selects any other transport cipher, or none |
| **TLS** | The dialer MUST verify the acceptor's certificate, either against the system trust store for the name it dialed, or against a fingerprint its operator pinned |

HOPE with AEAD is recommended because it **authenticates both ends** without any certificate. The transport keys are derived from the link password, so a server that does not know the password cannot produce a single valid frame, and the dialer detects an impostor on the first transaction it receives. TLS without certificate verification authenticates neither end, and the HOPE stream ciphers do not protect against an active attacker. Neither is acceptable for a link.

HOPE's login step necessarily exposes a password-derived MAC to whoever answers the dialer's connection, which permits an offline guessing attack on the link password. **Link passwords MUST therefore be randomly generated, with at least 128 bits of entropy**, and never chosen by a person. Implementations SHOULD generate them.

### Link Session Restrictions

Once bit 11 is confirmed, the session is a link and is no longer a user:

- The acceptor MUST NOT send the agreement (109) and MUST NOT wait for Agreed (121). The link is established by the login reply.
- The link session MUST NOT appear in Get User Name List (300), and MUST NOT cause Notify Change User (301) or Notify Delete User (302).
- **The acceptor MUST refuse every transaction on a link session other than those defined in this document**, regardless of the link account's access privileges. Ordinary transactions - chat, file listing, news, account administration - are not meaningful from a link and MUST be refused with an error, never silently processed.
- The dialer likewise MUST refuse any transaction from the acceptor that this document does not define.

---

## Transaction Types

All transaction IDs are allocated from the free 900-block (`0x0384`+). They are valid **only on a link session**; a server MUST refuse any of them arriving on an ordinary client session.

| ID | Hex | Name | Pattern | Description |
|---|---|---|---|---|
| 900 | `0x0384` | Link Hello | Notification | Version, features, epoch and the sender's own server group; first in each direction |
| 901 | `0x0385` | Link Snapshot | Notification | The sender's complete set of users exported to this peer |
| 902 | `0x0386` | Link User Update | Notification | One exported user appeared or changed |
| 903 | `0x0387` | Link User Gone | Notification | One exported user is no longer exported |
| 904 | `0x0388` | Link Chat | Notification | A public chat line |
| 905 | `0x0389` | Link Private Message | Request/reply | A private message, routed toward its recipient |
| 906 | `0x038A` | Link User Info | Request/reply | Fetch a user's info text from their home server |
| 907 | `0x038B` | Link Kick | Request/reply | Ask a user's home server to exclude their session from a server |
| 908 | `0x038C` | Link Ban | Request/reply | Ask a user's home server to exclude them from a server, for a period |
| 909 | `0x038D` | Link Unban | Request/reply | Lift a ban made with 908 |
| 910 | `0x038E` | Link Ping | Request/reply | Keepalive |
| 911 | `0x038F` | Link Close | Notification | The sender is closing the link, and why |
| 912 | `0x0390` | Link Servers | Notification | The complete set of servers reachable through the sender |
| 913 | `0x0391` | Link Server Update | Notification | A server became reachable through the sender, or changed |
| 914 | `0x0392` | Link Server Gone | Notification | A server, and all its users, are no longer reachable through the sender |

Transaction IDs 915–919 (`0x0393`–`0x0397`) are reserved for future link growth.

---

## Data Objects

Field IDs are allocated from the free `0x0630`-block.

| ID (hex) | Dec | Name | Type | Description |
|---|---|---|---|---|
| `0x0630` | 1584 | `DATA_LINK_VERSION` | UInt16 | Link protocol version. This document defines version `1` |
| `0x0631` | 1585 | `DATA_LINK_FEATURES` | UInt32 | Bitmask of [link features](#link-features) |
| `0x0632` | 1586 | `DATA_LINK_SERVER_NAME` | String | A server's name, for user info and operator display |
| `0x0633` | 1587 | `DATA_LINK_EPOCH` | Binary (8) | Random value the sender chose when it started |
| `0x0634` | 1588 | `DATA_LINK_USER_ID` | UInt16 | A user's ID as used by **the sender** of the transaction; opens each [user group](#the-user-group) |
| `0x0635` | 1589 | `DATA_LINK_TARGET_ID` | UInt16 | A user's ID as used by **the receiver** of the transaction |
| `0x0636` | 1590 | `DATA_LINK_MORE` | UInt16 | `1` when another part of the same snapshot follows |
| `0x0637` | 1591 | `DATA_LINK_REASON` | UInt16 | See [Link Reason Codes](#link-reason-codes) |
| `0x0638` | 1592 | `DATA_LINK_BAN_ID` | Binary (16) | Opaque handle for a ban, issued by the banned user's home server |
| `0x0639` | 1593 | `DATA_LINK_DURATION` | UInt32 | Ban duration in seconds; `0` = permanent |
| `0x063A` | 1594 | `DATA_LINK_SERVER_ID` | Binary (8) | A [server ID](#server-id). Opens each [server group](#the-server-group); inside a user group, the user's home server |
| `0x063B` | 1595 | `DATA_LINK_TAG` | String | A server's [tag](#tag) |
| `0x063C` | 1596 | `DATA_LINK_HOPS` | UInt16 | Links between the sender and the server described; `0` for the sender itself |
| `0x063D` | 1597 | `DATA_LINK_EXCLUDE` | Binary (8) | A server ID at which this user must not be shown. **Repeated** |
| `0x063E` | 1598 | `DATA_LINK_REQUESTER` | Binary (8) | The server ID on whose behalf a moderation request is made |

Field IDs `0x063F`–`0x064F` are reserved for future link growth.

Existing fields reused by this extension: `FieldError` (100), `FieldData` (101), `FieldUserName` (102), `FieldUserIconID` (104), `FieldChatOptions` (109), `FieldUserFlags` (112), `FieldOptions` (113), `FieldQuotingMsg` (214), and `DATA_COLOR` (`0x0500`) from [Colored Nicknames](Colored-Nicknames.md).

---

## Enumerations

### Link Features

`DATA_LINK_FEATURES` (`0x0631`). Each side offers the features its operator enabled for this peer, and the link uses the **intersection** of the two offers.

| Bit | Mask | Name | Meaning |
|---|---|---|---|
| 0 | `0x0001` | `LINK_FEATURE_PUBLIC_CHAT` | Public chat lines cross the link ([Link Chat (904)](#link-chat-904)) |
| 1 | `0x0002` | `LINK_FEATURE_PRIVATE_MESSAGES` | Private messages cross the link ([Link Private Message (905)](#link-private-message-905)) |
| 2 | `0x0004` | `LINK_FEATURE_USER_INFO` | User info requests cross the link ([Link User Info (906)](#link-user-info-906)) |
| 3 | `0x0008` | `LINK_FEATURE_TRANSIT` | Servers and users learned over this link may be relayed to the receiver's other transit links, and vice versa |

Bits 4–31 are reserved and MUST be sent as zero and ignored on receipt.

A feature is available between two servers only if every link on the path between them negotiated it. Each server applies its own links' features as it relays, so this follows without any server knowing the whole path.

User sharing, topology, moderation and the lifecycle transactions are not features: every link carries them. Ghosts always appear in the user list, because putting users in front of each other is what a link is for. Moderation is always honoured, because it is the condition on which operators trust each other enough to link.

**A server SHOULD offer `LINK_FEATURE_TRANSIT` by default.** Linking exists to bring communities together, and a network whose middle server does not relay is the three-windows problem again.

### Link Reason Codes

`DATA_LINK_REASON` (`0x0637`):

| Code | Name | Used in | Meaning |
|---|---|---|---|
| 0 | `OK` | replies | Success |
| 1 | `Disconnected` | 903 | The user left their home server |
| 2 | `NotExported` | 903 | The user is no longer exported over this link (became invisible, for example) |
| 3 | `UnknownUser` | replies | No user exported over this link has that ID |
| 4 | `RefusesMessages` | 905 reply | The recipient refuses private messages |
| 5 | `RateLimited` | replies | The sender exceeded a limit |
| 6 | `FeatureNotNegotiated` | replies | A link on the path did not negotiate the feature this needs |
| 7 | `UnknownBan` | 909 reply | No ban has that ID, or it was not made by the requester |
| 8 | `Unreachable` | replies | The server the request is for is not currently reachable |
| 9 | `Excluded` | replies | The sender or recipient is [excluded](#exclusion) at the other's server |
| 10 | `Banned` | 903 | The user was [banned from the network](#link-ban-908) and disconnected |
| 16 | `Shutdown` | 911 | The sender is stopping and expects to return; keep what it sent for the grace period |
| 17 | `Unlinked` | 911 | The sender's operator removed this link; discard what it sent now |
| 18 | `ProtocolError` | 911 | The sender received something it could not accept |
| 19 | `VersionUnsupported` | 911 | No common link protocol version |
| 20 | `Replaced` | 911 | A newer session for the same link has superseded this one |
| 21 | `Loop` | 911 | The link would connect servers that are already connected; see [Loop Prevention](#loop-prevention) |
| 22 | `TagConflict` | 911 | The link would join two servers with the same tag |
| 23 | `HopLimit` | 911 | The link would make the network deeper than the receiver allows |
| 24 | `Suspended` | 911 | The sender's operator has paused this link until further notice; discard what it sent now, and keep retrying at the slow pace |

A kick is not a departure reason: it takes effect as [exclusion](#exclusion), which hides a user at one server without removing them from the network. A ban is one, because a banned user is disconnected from the network.

---

## Transaction Semantics

Link transactions use standard Hotline framing (see [Hotline.md](Hotline.md)). Unlike the client/server protocol, **either end may originate any of them**:

- **Request/reply** (905–910): the originator sends with a unique non-zero task ID and the *is-reply* flag unset, and the other end replies with the same task ID and the *is-reply* flag set. Task IDs are scoped to their originator, so the two ends' task ID spaces are independent and may overlap. A failure reply has a non-zero error code, SHOULD carry `DATA_LINK_REASON`, and MAY carry `FieldError` text for operator logs.
- **Notification** (900–904, 911–914): sent with task ID `0` and no reply.

**Relayed requests.** A server that cannot answer a request itself - a private message for a user who is a ghost here, for example - forwards it over the link toward the server that can, with IDs translated, and answers the original request with the reply it receives. Each forwarding server SHOULD time out a forwarded request after 10 seconds and answer it with `Unreachable`.

All multi-byte integers are big-endian. Receivers MUST ignore unrecognised fields.

A receiver that cannot accept a transaction - an undefined type, a malformed group, an ID it was never given - SHOULD log it and drop it. It SHOULD close the link with `ProtocolError` only when it can no longer trust its picture of the peer's side of the network, for example a server or user update that could not be parsed. Dropping one unparseable chat line is better for everyone on both sides than splitting the network.

---

## Server Identity

### The Server Group

Servers are described by groups of fields, each opened by `DATA_LINK_SERVER_ID`:

| Field | Required | Notes |
|---|---|---|
| `DATA_LINK_SERVER_ID` | Yes | **First field of every group** |
| `DATA_LINK_TAG` | Yes | The server's own tag |
| `DATA_LINK_SERVER_NAME` | Yes | The server's own name |
| `DATA_LINK_HOPS` | Yes | `0` for the sender itself, otherwise the sender's distance to that server |
| `DATA_COLOR` (`0x0500`) | No | The color that server suggests for its users elsewhere ([Color](#color)) |

A group is complete, not incremental: it carries everything the sender holds for that server, and the receiver replaces what it had.

### Server ID

Every server has an 8-byte **server ID**, chosen at random the first time it links and kept permanently. It MUST NOT change when the server restarts, is renamed or changes its tag. Two servers with the same ID cannot be in the same network, which is how [loops](#loop-prevention) are detected, so a server MUST NOT derive its ID from anything another server could share (a hostname, a copy of a configuration file) and SHOULD generate it rather than accept one from its operator.

**Copying a server copies its ID.** An operator who sets up a second server by duplicating the first one's data, or who restores one server's backup onto another machine while the original still runs, ends up with two servers sharing an ID. Linking them, or placing both in the same network, is then refused as a loop. Implementations SHOULD keep the server ID somewhere an operator would not copy by accident, SHOULD give operators a command to generate a new one, and SHOULD say in the `Loop` log message that a duplicated server ID is one possible cause. Generating a new ID is harmless: peers see a new server, and nothing else depends on the old value.

### Tag

A tag is a server's own short name, chosen by its operator, and **unique within the network**. It is how users are told apart in display names (`bob@hl2`) and how other operators refer to the server in their configuration. A tag:

- MUST be 1–8 characters of printable ASCII, excluding `@` and whitespace;
- is compared case-insensitively;
- MAY be changed by its operator, which is announced like any other change to the server group.

A server MUST NOT accept a server group whose tag equals, case-insensitively, the tag of a different server ID it already knows, or its own. At establishment this closes the link with `TagConflict`; later, it ignores the update and logs it. Tag uniqueness is what lets a display name like `bob@hl2` mean the same server everywhere.

---

## Link Lifecycle

### Link Hello (900)

Each side MUST send Link Hello as its first link transaction, immediately after the login reply, and MUST NOT send any other link transaction until it has both sent and received Hello.

**Fields:** `DATA_LINK_VERSION`, `DATA_LINK_FEATURES` and `DATA_LINK_EPOCH` (all REQUIRED), followed by the sender's own [server group](#the-server-group) with `DATA_LINK_HOPS` = `0`.

- The link runs at the **lower** of the two versions. A side that cannot speak that version MUST send [Link Close](#link-close-911) with `VersionUnsupported`.
- The link's features are the bitwise AND of the two `DATA_LINK_FEATURES` values.
- On receiving Hello, each side checks the peer's server ID and tag against its own knowledge ([Loop Prevention](#loop-prevention), [Tag](#tag)), then sends [Link Servers](#link-servers-912).
- Each side sends its [Link Snapshot](#link-snapshot-901) only once it has received and accepted the peer's first complete Link Servers. A side that refuses the peer's server list (with `Loop`, `TagConflict` or `HopLimit`) has then sent none of its users over the link.

### Link Ping (910)

Request/reply with no fields. Each side SHOULD send a ping after 60 seconds without sending anything else, and SHOULD consider the link dead after three such intervals without receiving anything. A ping is answered whatever else is happening and is not subject to rate limits.

A ping is a link transaction, so neither side sends one before it has received Hello. A side whose peer has not sent Hello within about 30 seconds of the login reply SHOULD send Link Close with `ProtocolError` and end the session.

### Link Close (911)

**Fields:** `DATA_LINK_REASON` (REQUIRED), `FieldError` (optional, text for the peer's log).

The sender closes the connection after sending it. The reason decides what the receiver does with everything learned over the link:

- `Shutdown`: keep it for the [grace period](#interruption-and-resynchronisation), as for an interruption.
- Any other reason: discard it immediately.

`Suspended` is an operator pausing a link without removing it, for maintenance or during an incident on the other server. The paused side refuses the link until its operator resumes it, closing every attempt with `Suspended`. A dialer receiving it SHOULD keep retrying at its slowest pace, because the pause may end at any time and the dialer has no other way to learn that it has. A suspension is not a refusal because of the network's shape, and not an `Unlinked`: unlike `Unlinked`, it MUST NOT stop the dialer, or a resume would never reconnect. An implementation SHOULD keep a suspension across restarts, and SHOULD make it visible to its operator: a link that silently re-forms after a restart, mid-incident, is the surprise suspension exists to prevent.

A dialer receiving `Unlinked`, `VersionUnsupported` or `Replaced` MUST NOT reconnect automatically. Each needs an operator to act.

`Loop`, `TagConflict` and `HopLimit` are refusals because of the network's shape, and a dialer receiving one MUST NOT treat it as final. It SHOULD log the refusal as an error and retry with its ordinary backoff, growing to the ceiling. Two reasons make this necessary:

- **Links that come up at the same moment can each be refused because of the other.** A dialer that gave up would leave the network split until an operator noticed (see [Loop Prevention](#loop-prevention)).
- **The refusal can stop being true.** The server that made a link redundant may leave the network, or the server whose tag collided may rename.

A link that is truly redundant settles at the slow pace and stays visible in its operator's log.

### Interruption and Resynchronisation

A link that drops without `Link Close`, or closes with `Shutdown`, is **interrupted**, not ended. The receiving server keeps the servers and ghosts it learned over that link for a grace period (RECOMMENDED: 60 seconds), so a brief outage is not shown to every client as half the network leaving and coming back.

- **During the grace period** the server tells nobody: it does not remove the ghosts locally, and it does not send Server Gone or User Gone over its other links. Requests addressed through the interrupted link fail with `Unreachable`. A server MAY set the away flag on the affected ghosts for the duration.
- The dialer reconnects with backoff: first attempt promptly, then increasing to a ceiling of a few minutes, with jitter.
- On reconnection, each side sends its full server list and snapshot as usual, and the receiver reconciles them against what it kept:
  - **Same epoch** in the peer's Hello as before the interruption: the peer's user IDs still mean the same users. Keep ghosts whose ID appears in the snapshot (relaying an update only if something about them changed), remove those whose ID does not, and add the new ones.
  - **Different epoch**: the peer restarted, and its user IDs now name different people. Every ghost learned over that link is stale. The receiver SHOULD remove all of them and add the snapshot's users afresh. It MAY instead re-use a ghost for an incoming user with the same home server, name and icon, which avoids showing a reconnected user leaving and rejoining. It MUST update the ID mapping when it does.
  - Servers are reconciled by server ID: those in the new list are kept or updated, and those missing are gone.
- When the grace period expires without reconnection, the server removes the ghosts and relays [Link Server Gone](#link-server-gone-914) for every server learned over the link. It MAY post a notice in public chat that the link was lost.

A split that outlasts the grace period is announced honestly: every client sees the users on the far side leave, and sees them all arrive again when the link returns. The same happens when a server first joins a network. This is the netsplit familiar from IRC and from linked game realms, and it is expected behaviour, not something to suppress. It is the user list telling the truth about who can be reached.

Because each server absorbs interruptions on its own links, a brief outage between two servers is invisible to the rest of the network. A server re-using ghost IDs across a peer's restart keeps the outage invisible beyond itself too.

### One Link per Peer

- **Only one side dials.** Each server's configuration for a peer says either that it dials that peer or that it accepts from that peer, never both. A server cannot always tell from its configuration that an accepting entry and a dialing entry are the same peer (the entries name a local account and an address, not a server ID), so it is not required to reject such a configuration. Either mistake (both sides dialing, or one side holding both entries) produces two links between the same pair; the second is closed with `Loop` (see below), so it fails safe, but it fails noisily.
- **Newest wins on the same account.** If a link login arrives for a link account that already has a live link session, the acceptor MUST send `Link Close` with `Replaced` on the old session and keep the new one. The old session is almost always a half-open connection the dialer has already given up on, and refusing the new one would leave the link down until the old one timed out. The acceptor SHOULD treat the takeover as a reconnection for [resynchronisation](#interruption-and-resynchronisation), which skips the loop check against the session it replaces.

---

## Network Topology

Each server keeps a table of every server it knows: its own entry, plus every server learned over each link, with the link it was learned over. In a tree each server is learned over exactly one link, and that link is the route to it.

### Link Servers (912)

Notification carrying the complete set of servers reachable through the sender, **excluding** any it learned over this link, as repeated [server groups](#the-server-group). The sender itself is not included (it was described in Hello). Sent after Hello on every connection, including reconnections.

**Fields:** repeated server groups; `DATA_LINK_MORE` = `1` on every part except the last.

The servers reachable through the sender over this link are those it learned over its other links that negotiated `LINK_FEATURE_TRANSIT`, and only when this link negotiated it too. Without transit on this link, the list is empty, and the peer sees only the sender's own users.

`DATA_LINK_HOPS` in each group is the sender's own distance to that server. The receiver adds one.

### Link Server Update (913)

Notification carrying one server group: a server newly reachable through the sender, or a change to one already announced (name, tag, color, distance). Relayed onward under the same transit rules.

**A server is always announced before its users.** A sender MUST NOT send a user group whose home server it has not already announced on that link: in Hello, in Link Servers, or in a Link Server Update. When a server becomes reachable mid-session, as when a new server joins somewhere else in the network, the sender sends its Server Update first and its users' User Updates after. Because each link is one ordered stream, this guarantees the receiver knows the home server of every user it is given. Without the guarantee, the rule under [The User Group](#the-user-group) would silently drop the users of every server that joins after a link is established. Conversely, Server Gone is sent *instead of* User Gone for that server's users, never after them.

### Link Server Gone (914)

Notification that a server is no longer reachable through the sender.

**Fields:** `DATA_LINK_SERVER_ID` (REQUIRED).

**It implies the departure of every user whose home server is that server.** The receiver removes those ghosts, announcing Notify Delete User (302) for each to its clients, and relays Server Gone onward. It does not send User Gone for each of them. When a large part of the network becomes unreachable, the departure crosses each link as one transaction per server rather than one per user.

### Loop Prevention

A loop exists when a server would learn of the same server over two links. A server MUST check, for **every server group it receives** (in Hello, Link Servers, or Link Server Update):

- that its server ID is not the receiver's own; and
- that its server ID is not already known over a *different* link.

If either check fails during establishment (in Hello or the first Link Servers), the receiver MUST close the new link with `Loop`, before any users have been exchanged. If a check fails later, on a Link Server Update, the receiver MUST close **the link that delivered the update** with `Loop`.

Links that come up at the same moment can close a loop that no single server sees forming, and each server that detects it breaks a different link, which can split the network further than necessary, briefly leaving a server on its own. This is why `Loop` is not a final refusal ([Link Close](#link-close-911)). The dialers retry with backoff and jitter, the first link to complete establishment wins, and every later attempt that would close a loop is refused at establishment, so the network settles as a tree that still joins every server. Which link ends up refused depends on timing, not on configuration. Operators see it in their logs, retrying, with the reason `Loop`. The cure is to remove the redundant link.

A server SHOULD also enforce a maximum network depth (RECOMMENDED: 8 hops). A server group whose distance, after adding one, would exceed it closes the link with `HopLimit` during establishment, and is ignored and logged afterwards, as are the users homed on it. The bound keeps a misconfigured chain from growing indefinitely, and it makes relayed request latency predictable.

### Routing

Requests that must reach a particular server travel hop by hop:

- **Requests about a user** (905, 906, 907, 908) are addressed by user ID. Each server knows whether that ID is a local user, and if not, which link the ghost came from. It forwards the request there with the ID translated.
- **Requests about a server** (909, which names the ban's home server) are routed by server ID through the topology table.

No server needs the whole path. It needs only the next hop, and that is always the link the target was learned over.

---

## User Sharing

### Who Is Exported

Over each link, a server exports:

1. **its own local users**: every local session that an ordinary, unprivileged client would see in Get User Name List (300), excluding link sessions, sessions hidden from the classic user list (such as [pure messenger sessions](Capabilities-Messaging.md#capability-bits)), and users invisible to unprivileged clients; and
2. **if this link negotiated `LINK_FEATURE_TRANSIT`**, every ghost it learned over its *other* transit links. Ghosts learned over this link are never sent back over it.

A local session is exported from the moment it would be announced to local clients with Notify Change User (301), after login and agreement, not before. It stops being exported when it would be removed from their lists. A session that becomes hidden (going invisible, for example) is withdrawn with [Link User Gone](#link-user-gone-903) and reason `NotExported`, and re-exported with [Link User Update](#link-user-update-902) when it becomes visible again.

Users [excluded](#exclusion) somewhere are still exported, with their exclusions, so that every server between them and the excluding server can carry them past it.

### The User Group

| Field | Required | Notes |
|---|---|---|
| `DATA_LINK_USER_ID` | Yes | The user's ID as used by the sender. **First field of every group** |
| `DATA_LINK_SERVER_ID` | Yes | The user's home server |
| `FieldUserName` (102) | Yes | The user's own name, UTF-8. Never a display name with a tag added |
| `FieldUserIconID` (104) | Yes | The user's icon ID, unchanged. Icon IDs are a long-settled shared vocabulary; a receiver shows whatever its clients show for that ID, and a client that does not display custom icons falls back as it would for a local user |
| `FieldUserFlags` (112) | Yes | Filtered as described under [User Flags](#user-flags) |
| `DATA_COLOR` (`0x0500`) | No | The user's own nickname color, if they set one |
| `DATA_LINK_EXCLUDE` | No, repeated | Servers at which this user must not be shown ([Exclusion](#exclusion)) |

In a transaction carrying several groups, every field from one `DATA_LINK_USER_ID` up to the next (or the end of the transaction) belongs to that user.

**A relaying server passes the home server, name, icon, color and exclusions on unchanged.** It applies its own [flag](#user-flags) rules and substitutes its own ID for the user.

Senders MUST NOT send any of the user's IP address, login, account name, account privileges or connection details. No server in the network has any use for them, and a network that never carries them can never leak them.

**A group is complete, not incremental.** It carries everything the sender currently holds for that user, and the receiver replaces what it had. An absent `DATA_COLOR` means the user has no color, and an absent `DATA_LINK_EXCLUDE` means no exclusions, never "unchanged". This is the rule the messaging extension adopted for [roster entries](Capabilities-Messaging.md#an-entry-group-is-complete-not-incremental), for the same reason: a flat field list cannot say "unchanged".

A receiver MUST drop a user group whose home server is not a server it learned over the same link, or is itself. A peer may only present users from the part of the network that lies behind it. Since a conforming sender [announces every server before its users](#link-server-update-913), a dropped group means a faulty or dishonest peer, and the receiver SHOULD log it.

### Exclusion

Exclusion is how a [kick](#link-kick-907) takes effect. A user's group lists every server at which that user must not be shown for the rest of their session. **Only the user's home server sets exclusions**, from the kicks it holds, and every other server passes them on unchanged. (A [ban](#link-ban-908) needs no exclusion: the banned user is disconnected, so there is nobody left to show.)

A server that finds its own server ID among a user's exclusions:

- MUST NOT show that user: no entry in Get User Name List (300), no Notify Change User (301);
- MUST NOT display their public chat lines, and MUST refuse private messages from them or to them with `Excluded`;
- **MUST still relay them**, their lines and their messages, to its other links as usual. The kick was made by this server for this server, and the servers beyond it have not kicked anybody.

A server typically keeps an excluded user as a hidden ghost: allocated and mapped like any other, so it can be relayed, but never listed.

### Link Snapshot (901)

Notification carrying the complete set of users exported over this link, as repeated [user groups](#the-user-group). Sent on every connection, including reconnections, once the sender has received and accepted the peer's first complete [Link Servers](#link-servers-912) (see [Link Hello](#link-hello-900)). The sender's own Link Servers always comes first.

**Fields:** repeated user groups; `DATA_LINK_MORE` = `1` on every part except the last.

A sender MAY split a large snapshot across several transactions. The receiver MUST treat all parts, up to the one without `DATA_LINK_MORE`, as one snapshot, and MUST NOT reconcile until it has the last. A sender MUST NOT send any other user or server transaction between the parts.

An empty snapshot (one transaction, no groups, no `DATA_LINK_MORE`) is valid.

### Link User Update (902)

Notification carrying exactly one [user group](#the-user-group): a user newly exported over this link, or a change to one already exported (name, icon, flags, color or exclusions).

The receiver creates a ghost if it has none for that ID, or updates the existing one. It announces the result to its local clients with Notify Change User (301), unless the user is excluded here, and relays the update onward.

A change in exclusions can make a ghost appear or disappear locally. Appearing is announced with 301 and disappearing with 302, while the relay onward is always an update.

### Link User Gone (903)

Notification that one user is no longer exported over this link.

**Fields:** `DATA_LINK_USER_ID` (REQUIRED), `DATA_LINK_REASON` (REQUIRED).

The receiver removes the ghost, announces Notify Delete User (302) if it was shown, and relays the departure onward. The reason is for operators' logs. Local clients see an ordinary departure whatever it was.

---

## Presenting Ghosts

### Display Name

By default a ghost is shown under **the user's own name, unchanged**, and is distinguished by [color](#color). A server SHOULD offer its operator an option to append the home server's [tag](#tag) instead, giving `bob@hl2`.

Two rules apply in either mode:

- **A ghost's name MUST NOT duplicate another user's name** in the receiving server's user list, compared case-insensitively. If it would, the server appends the tag to that ghost (`bob@hl2`), whatever the option says. Local users always keep their own names, and it is always the ghost that is tagged:
  - If a local user later takes a name that an untagged ghost already has, the server tags the ghost and announces the change with 301.
  - If two ghosts collide, the one announced later is tagged.
  - A ghost tagged this way keeps its tag until its own name next changes.
- **When the tag is appended and the result exceeds the server's name length limit**, the server shortens the name part, never the tag. A truncated name is unremarkable, whereas a truncated tag hides where the user is from.

Display names are local. Each server decides its own, and relays the user's own name, never its display name. A user tagged on one server because of a name collision there is untagged everywhere else.

The collision rule matters most for clients that cannot show color. Without it, a ghost could take exactly the name of a local user, including a local administrator, and appear identical to them.

### Color

Every ghost is colored by its **home server**. The color a server uses for another server's users is, in order of preference:

1. a color the receiving server's operator configured for that server's tag;
2. the color that server suggests in its [server group](#the-server-group);
3. a color the receiving server derives from the tag.

Using the home server's suggestion by default means a server's users appear in the same color across the network. Colors are delivered through [Colored Nicknames](Colored-Nicknames.md) under that extension's usual rules, so only clients that support colored nicknames see them.

The server's color takes precedence over the user's own `DATA_COLOR`. A ghost whose name is not tagged is identified only by its color, and a user free to pick their own color could pick the local one. A server MAY use the user's own color when it is appending the tag, because the tag then identifies the ghost.

Clients without colored-nickname support see untagged ghosts as ordinary users. The [user info](#user-info) text still names the ghost's home server. Operators whose communities mostly use such clients may prefer the tag option.

### User Flags

`FieldUserFlags` (112) on a link. Each relaying server applies these rules again as it relays:

| Bit | Meaning | On the link |
|---|---|---|
| 0 | Away | Passed through |
| 1 | Admin | **Every sender MUST clear it, and every receiver MUST clear it again.** Administrative status never crosses a link |
| 2 | Refuses private messages | Passed through. A server MUST also set it on every ghost learned over a link that did not negotiate `LINK_FEATURE_PRIVATE_MESSAGES`, both when showing it and when relaying it |
| 3 | Refuses private chat | A server MUST set it on every ghost. Private chat with a ghost is not supported in this version |

Applying bit 2 hop by hop means a ghost reaches every server already marked "refuses private messages" if any link on its path cannot carry them, and a classic client greys out the action before the user tries it.

### Transactions Naming a Ghost

A local client may address a ghost's user ID in any transaction that names a user. A server MUST handle each as follows:

| Transaction | Handling |
|---|---|
| Send Instant Message (108) | Translated to [Link Private Message (905)](#link-private-message-905) |
| Get Client Info Text (303) | Translated to [Link User Info (906)](#link-user-info-906) |
| Disconnect User (110) | Translated to [Link Kick (907)](#link-kick-907), or with the ban options to [Link Ban (908)](#link-ban-908). Requires the caller's ordinary disconnect privilege on the receiving server |
| Invite to New Chat (112), Invite to Chat (113) | Refused with an error |
| Anything else naming a user ID | Refused with an error |

The last row is the important one. **Unknown transactions on a ghost MUST fail closed.** A ghost is not a connection. Any handler that assumes it can write to a user, start a transfer to them or inspect their account must never be reached with a ghost's ID. A server that falls through to such a handler fails unpredictably; a server that refuses is merely incomplete.

**Local privileges are checked first.** Before translating anything, the server applies the privilege check the transaction would get with a local target: Send Private Message for 108, Get Client Info for 303, Disconnect User for 110, and Send Chat for a public chat line. A user who cannot do something to a local user cannot do it to a ghost either, and a refused request never reaches a link.

### Ghosts and Other Extensions

A ghost exists only in the classic user list. It has no account, no Login and no session capabilities, so:

- **Messaging.** A ghost MUST NOT appear in any [messaging](Capabilities-Messaging.md) roster, Find User or User Search result. Messaging addresses accounts by Login, which a ghost does not have. No messaging transaction can name a ghost, and a server MUST NOT treat a ghost's display name as a Login.
- **Voice and video.** A ghost is never a member of a voice room, including public chat's. Voice room status lists local participants only.
- **Chat history.** Lines from ghosts are stored and replayed like local lines, with the ghost's display name and icon and the admin flag clear (see [Public Chat](#public-chat)).
- **Inline media.** Attachments do not cross links (see [Link Chat](#link-chat-904)).
- **Colored nicknames.** Handled as described under [Color](#color).

---

## Public Chat

When `LINK_FEATURE_PUBLIC_CHAT` is negotiated, public chat (chat ID `0`) is shared across the link, and therefore, link by link, across the network: a line said on any server is seen on every server connected to it by links that all carry public chat.

### Link Chat (904)

Notification carrying one public chat line.

**Fields:** `DATA_LINK_USER_ID` (REQUIRED, the speaker, as the sender identifies them), `FieldData` (REQUIRED, the text as the speaker sent it, UTF-8), `FieldChatOptions` (optional, `1` for an emote, as in Send Chat (105)).

**Originating.** When a local, exported user sends a public chat line that the server accepts and broadcasts locally, the server sends Link Chat over each link that negotiated the feature. It sends the **raw text** the user typed, never the formatted line its own clients receive.

**Relaying.** When a server receives Link Chat over a link, it relays it over each of its other links that negotiated both public chat and transit, provided the link it arrived on negotiated transit. Each relayed copy names the speaker by that server's own ID for them.

A server MUST NOT originate or relay Link Chat for:

- lines from users not exported over the outgoing link;
- server messages, administrator broadcasts, or lines injected by bridges;
- chat in any room other than public chat;
- inline media or other extension content attached to a line. The text is sent; the attachment is not. A line that consists only of an attachment, with no text, is not sent at all, rather than sent empty.

**Receiving.** The receiver validates that the speaker is a current ghost from that link and that the line is within its [limits](#limits), then **formats and delivers it exactly as it would a line from a local user**, with the ghost's display name. It does this unless the speaker is [excluded](#exclusion) here, in which case it only relays. Formatting at the receiver is what makes remote lines look like local ones: the same layout, the same encoding conversion for each client and the same chat history. It also means no server needs to know how another formats chat.

**Local filtering affects local display only.** A server may filter chat with its own rules: word filters, anti-spam, plugins. Applied to a line from a ghost, such a filter decides whether *this* server shows the line, and never whether the line is relayed onward. Every server applies its own filters to what it shows, which is the same principle as [exclusion](#exclusion). Applied to a line from a local user, a filter that rejects the line rejects it outright, as today: a rejected line is never accepted, so it is never originated over a link.

A receiver SHOULD record ghost lines in its chat log and chat history as it records local lines. Each server's history is therefore the chat **it saw**: lines from before a link existed, or from while it was split from part of the network, are not backfilled. A receiver MAY pass ghost lines to its own bridges (an IRC bridge, for instance), which are local outputs rather than links.

Ordering is preserved along each path. Lines from different servers may interleave slightly differently on different servers, because they cross in transit. This is inherent, not a fault.

A server that receives Link Chat over a link that did not negotiate public chat MUST drop it.

---

## Private Messages

When `LINK_FEATURE_PRIVATE_MESSAGES` is negotiated on every link of the path, users on different servers can exchange private messages.

### Link Private Message (905)

Request/reply, [relayed](#routing) hop by hop toward the recipient.

**Request fields:** `DATA_LINK_USER_ID` (REQUIRED, the sender, as the sending server identifies them), `DATA_LINK_TARGET_ID` (REQUIRED, the recipient, as the receiving server identifies them), `FieldData` (REQUIRED, the message), `FieldQuotingMsg` (214, optional), `FieldOptions` (113, optional, passed through as in Send Instant Message (108), so an automatic response stays one).

**Reply fields:** `DATA_LINK_REASON`: `OK`, or on failure `UnknownUser`, `RefusesMessages`, `Excluded`, `RateLimited`, `FeatureNotNegotiated` or `Unreachable`.

**Originating.** A local user sends Send Instant Message (108) to a ghost's ID. The server sends Link PM over the link that ghost came from, naming the sender by their local ID and the target by the ID the peer exported them under. The server MUST refuse the 108 locally, without sending anything, when the sender is not exported over that link. An invisible user has no ghost anywhere else to be the sender of the message, and must not be able to message out of a server where nobody can see them. It SHOULD also refuse locally when the sender is excluded at the target's home server, since the home server would refuse anyway.

**Receiving.** The receiving server checks that:

1. the sender is a current ghost from that link (otherwise `UnknownUser`);
2. **the target is a user this server exported over that link** (otherwise `UnknownUser`). A peer can address only users it has been shown, which stops a peer from reaching an invisible user by guessing their ID.

If the target is a ghost here, the server relays the request onward over the link that ghost came from, with both IDs translated, provided that link negotiated private messages (otherwise `FeatureNotNegotiated`). It then answers with the reply it receives.

If the target is local, the server checks that neither party is [excluded](#exclusion) at the other's home server as far as it can tell (otherwise `Excluded`), and that the target accepts private messages (otherwise `RefusesMessages`). It then delivers Server Message (104) to the target as from the sender's ghost: the ghost's user ID and display name. The target replies with an ordinary 108 to that ghost, which is routed back the same way. Neither client can tell that the conversation crossed servers.

The originating server SHOULD reflect a failure back to the local user as a failed reply to their 108, with error text naming the reason.

---

## User Info

### Link User Info (906)

Request/reply, [relayed](#routing) hop by hop to the user's home server. Used when a local client requests Get Client Info Text (303) for a ghost, and every link of the path negotiated `LINK_FEATURE_USER_INFO`.

**Request fields:** `DATA_LINK_TARGET_ID` (REQUIRED, the user, as the receiving server identifies them).

**Reply fields:** `FieldData` (the info text, UTF-8), or a failure with `DATA_LINK_REASON`.

The home server MUST build the text **it would show an ordinary, unprivileged client**, whatever privileges the requester has on its own server. In particular it MUST NOT include the user's address, login or account details. An administrator on one server is not an administrator on another, and is not entitled to what the home server shows its own administrators. Relaying servers pass the text through unchanged.

The server that received the 303 MUST prefix the text with a line naming the ghost's home server (from its server group) before returning it to its client. If the request cannot be answered (feature missing on the path, `Unreachable`, no reply within a few seconds), it MUST return a text of its own that names the home server and says nothing more. This is the one identification every client can display, whether or not it supports colors.

---

## Moderation

Moderation is part of every link, and the people who moderate are trusted network-wide. **A user who holds the disconnect privilege on any server in the network is a moderator of the network**:

- A **kick** removes a ghost from the moderator's own server for the rest of that session.
- A **ban** removes the person from the whole network.

Both are carried out by **the user's home server**, because only the home server can make them hold. It knows the user's account and address, it holds their connection, and it is the server that would otherwise export them again the moment they reconnect. No other server learns either identifier.

A moderator acts through the ordinary tools: Disconnect User (110) on a ghost (with the ban options for a ban), or the server's administrative interface. The privilege required is the moderator's ordinary disconnect privilege **on their own server**. A user's privileges on their home server, including any protection from being disconnected, do not shield them from a moderator elsewhere. Granting someone the disconnect privilege on a linked server makes them a moderator of the network, and operators should grant it with that in mind.

Kick and ban requests are [relayed](#routing) hop by hop toward the user's home server. They carry `DATA_LINK_REQUESTER`, set by the server where the moderator acted, and every relaying server passes it on unchanged. The home server MUST accept the requester's identity as relayed. This is the [trust model](#trust-model) at work: the requester was vouched for by the servers in between.

A ban a server places on one of its own local users needs nothing from this section. The user can no longer connect to their home server, so they are gone from the network too.

### Link Kick (907)

Request/reply. The requester asks the user's home server to remove one session from the requester.

**Request fields:** `DATA_LINK_TARGET_ID` (REQUIRED, the user, as the receiving server identifies them), `DATA_LINK_REQUESTER` (REQUIRED), `FieldData` (optional, a reason for the home server's log).

**Reply fields:** `DATA_LINK_REASON`: `OK`, `UnknownUser` or `Unreachable`.

- The requesting server hides the ghost immediately, without waiting for the reply, and keeps it hidden until the exclusion arrives or the user is gone.
- The home server MUST add the requester to that session's [exclusions](#exclusion) for as long as the session lasts, and send the updated user group. The user **stays connected to their home server** and visible everywhere else.
- The home server SHOULD tell the user, with a server message naming the requesting server, so a kicked user knows why that server's users vanished.

A kick is the proportionate tool: it ends one person's presence on one server without affecting how anyone else sees them.

### Link Ban (908)

Request/reply. The requester asks the user's home server to ban this person from the network, for a period.

**Request fields:** `DATA_LINK_TARGET_ID` (REQUIRED), `DATA_LINK_REQUESTER` (REQUIRED), `DATA_LINK_DURATION` (REQUIRED; `0` = permanent), `FieldData` (optional, a reason).

**Reply fields:** `DATA_LINK_BAN_ID` and `DATA_LINK_REASON` = `OK`; or a failure with `UnknownUser` or `Unreachable`.

- The requesting server hides the ghost immediately, without waiting for the reply.
- **If the ban fails** (`Unreachable`, `UnknownUser`, or no reply), the requesting server MUST keep that session hidden for the rest of the session, as though it had been [kicked](#link-kick-907), and MUST tell the moderator that the ban was not applied. It MUST NOT queue the ban for later delivery. A ban is applied against the person as their home server sees them at the moment it acts, and a queued ban could land on whoever holds that user ID by then. The moderator can ban again once the home server is reachable.
- The home server MUST ban the person **exactly as if its own operator had**. It disconnects every matching session and refuses matching logins for the duration. Their departure crosses the network as [Link User Gone](#link-user-gone-903) with reason `Banned`, and every server removes them.
- The home server MUST record the ban against **the most stable identifier it has for that person**: the account, for an account one person uses, and the connection address for a shared account such as `guest`. It records the ban together with the requester's server ID, the reason and the duration. It MUST persist the ban across restarts, and MUST NOT reveal the identifier to anyone.
- The home server MUST record the ban where its own operator can see it, with the requesting server's tag and name. A home operator needs to know why one of their regulars can no longer connect.
- The home server SHOULD tell the user, in the disconnect message, that they were banned from the network and by which server.
- The home server returns an opaque, unguessable `DATA_LINK_BAN_ID`. The requester stores it with the home server's ID and what it knew at the time (the ghost's display name, the reason, the time and the duration). That is what its operator sees when listing bans.

A ban is enforced at the user's home server, so it holds against the person *as that server knows them*. Someone banned this way who connects to a different server in the network is a new person to that server, which has no means of recognising them, and moderators there deal with them as they would with any newcomer. Recognising a banned person at every server's door would require their account or address to cross links, which this version deliberately never does (see [Future Work](#future-work)).

### Link Unban (909)

Request/reply, routed by server ID to the ban's home server.

**Request fields:** `DATA_LINK_BAN_ID` (REQUIRED), `DATA_LINK_SERVER_ID` (REQUIRED, the home server that issued the ban), `DATA_LINK_REQUESTER` (REQUIRED).

**Reply fields:** `DATA_LINK_REASON`: `OK`, `UnknownBan` or `Unreachable`.

The home server MUST honour an unban from the requester that made the ban, and answers `UnknownBan` to any other requester. The home server's own operator MAY also lift any ban on its users through its local tools, because it is a ban on their server. When they do, the home server SHOULD tell nobody: the requester's stored ban ID simply stops matching anything, and a later unban from the requester answers `UnknownBan`. Matching logins are accepted again from the moment a ban is lifted. An expired ban is removed by the home server without any message.

---

## Limits

A receiver applies its own limits to everything arriving over a link, and MUST NOT rely on the sender's. A peer's flood control protects the peer; the receiver protects itself.

- **Ghosts per link.** A server SHOULD bound the number of ghosts it holds from each link, counting every user behind it, relayed or not. Users beyond the bound are not represented, and traffic naming them is dropped. The server SHOULD log when the bound is reached.
- **Network depth.** See [Loop Prevention](#loop-prevention).
- **Chat and messages.** A server SHOULD rate-limit Link Chat and Link Private Message per ghost and per link, at the same thresholds it applies to local users, and MUST bound their length as it does for local users. A line dropped by a limit is not relayed either.
- **Server capacity.** Ghosts MUST NOT count toward the server's connection limit or per-address limits. They consume user IDs and nothing else.
- **Tracker listings and the info port.** A server SHOULD NOT count ghosts in the user count it reports to trackers, or in `users.connected` on the [info port](Hotline-Info-Port.md). A user is connected to one server, and counting them on every server in the network inflates every listing. A server MAY report the number of ghosts it currently shows as `users.linked` on the info port, and that it links and how many ghosts it shows as `SUPPORTS_SERVER_LINKING` and `LINKED_USERS` in a [v3 tracker registration](Tracker-Protocol-v3.md).

---

## Security Considerations

- **The link password is the keys to the user list.** Anyone holding it can place users on the peer's server under any name, attributed to any server behind them. It MUST be random ([Protecting the Link](#protecting-the-link)), SHOULD be unique per peer, and SHOULD be rotated when an operator leaves. Closing a link is always unilateral, so revoking it requires nothing from the other side.
- **Peer authentication is mutual** on every link. With HOPE AEAD, both servers prove knowledge of the link password before any link traffic is accepted.
- **Trust is transitive, by design.** A server cannot verify what a peer says about servers behind it. A dishonest server in the middle of a network could invent users from a server it relays, or drop a ban request. This is the price of relaying, and it is why the [trust model](#trust-model) treats joining a network as vouching for it, and why transit is a per-link choice.
- **Identity assertions are scoped.** A peer can present only users homed behind it, can speak only for users it exported, and can address only users it was shown. Every transaction naming a user or a server is checked against those sets.
- **Privilege never crosses.** The admin flag is cleared by every sender and every receiver, ghosts hold no access privileges, and user info is built as for an unprivileged requester.
- **No personal data crosses.** No address, login or account detail is ever sent. Bans work without them because home servers enforce them. A user's exclusions do reveal which servers have kicked them to every server they are relayed through; operators who link have accepted that.
- **Impersonation.** The [display name](#display-name) rules prevent a ghost from taking a local user's exact name. They cannot prevent look-alike names (`adm1n`, Unicode confusables), any more than a single server can between its own users. Color and the user-info prefix remain the reliable identification.
- **Content is visible to every server on its path**, and to anyone who reads those servers' chat logs. Whether and how to tell users that the server is part of a network (in the agreement, for instance) is left to each operator.
- **Network membership is not published.** Which servers a server is linked with, and the tags and IDs of the servers in its network, are known to the network's operators and its users, not to the outside. A server MUST NOT publish them outside the link: not to trackers, not on the info port, and not in any other public listing. That a server links, and how many linked users it shows, MAY be published (see [Limits](#limits)). A network's membership maps the trust relationships between its operators, and an attacker looking for a way in should not get it for free.
- **Transactions on a ghost fail closed** ([Transactions Naming a Ghost](#transactions-naming-a-ghost)). This is the property most likely to regress as a server gains features, and it deserves a test of its own.

---

## Client Behaviour

No client changes are required. Clients see ghosts as users.

Clients MAY improve the experience:

- Clients that support [Colored Nicknames](Colored-Nicknames.md) show each server's users in that server's color with no work specific to this extension.
- Clients SHOULD honour the refuse-private-messages and refuse-private-chat [user flags](#user-flags), which servers set on ghosts to mark actions that cannot work.

## Server Behaviour

A conforming server:

1. Confirms `CAPABILITY_SERVER_LINK` only for a configured link account, only alongside `CAPABILITY_TEXT_ENCODING`, and only on a session protected as described in [Protecting the Link](#protecting-the-link).
2. Hides link sessions from the user list, skips the agreement for them, and refuses every non-link transaction on them, regardless of the link account's privileges.
3. Keeps a permanent random server ID and a network-unique tag, and refuses links that would create a loop, duplicate a tag, or exceed its depth limit.
4. Exports its own visible local users, and relays ghosts and servers between transit links, never back over the link they came from. It announces every server before any of its users, and never sends addresses, logins or privileges.
5. Sends Hello first and nothing before it, then its server list, and its users only once it has accepted the peer's server list. It closes a link whose peer never sends Hello, and one that sends server or user state it cannot parse.
6. Presents ghosts with an allocated user ID, their home server's color, flags filtered as specified, and a display name that never duplicates another user's.
7. Validates every incoming reference. A user must be homed behind the link they arrived on, a sender must be a current ghost from that link, and a target must be a user exported over that link.
8. Formats remote chat lines locally, from raw text, as it would a local line.
9. Relays private messages, user-info requests and moderation requests hop by hop, translating IDs, and passes replies back.
10. Applies local privilege checks before translating anything, refuses every transaction naming a ghost that this document does not translate, and keeps ghosts out of messaging, voice and every other extension.
11. As a home server: honours kick, ban and unban requests for its own users from any server in the network. It enforces kicks through exclusions and bans by disconnecting and refusing the person as its own ban would, persists bans, shows them to its operator, and never reveals the banned identifier.
12. Hides users excluded at itself while still relaying them.
13. Absorbs link interruptions for a grace period, reconciles against the next snapshot using the epoch, and relays Server Gone when the grace period expires.
14. Applies its own limits to link traffic, and excludes ghosts from connection limits and tracker counts.
15. Publishes nothing about its network's membership: not which servers it links with, nor their tags or IDs.

---

## Future Work

Each item below would be a new [link feature](#link-features) bit, or a new transaction in the reserved range, negotiated per link, so a server that does not implement it continues to interoperate.

- **Redundant links.** Standby links between servers that are already connected, held idle while the tree is whole and activated when it splits. This heals a broken link without an operator.
- **Private chat across links.** Inviting a ghost into a private chat, with the chat hosted on, and moderated by, the server where it was created.
- **Broadcasts.** Delivering an administrator broadcast (355) to other servers' users, with those servers' consent.
- **Inline media and icons.** Carrying [inline media](Capabilities-Inline-Media.md) and [GIF icons](GIF-Icons.md) across links. Both need content to be fetched from, or cached away from, the home server.
- **Voice and video** in shared public chat.
- **File transfer** between users on different servers.
- **Messaging.** Bringing the [instant messaging](Capabilities-Messaging.md) roster and presence across links, so users on different servers can be friends.
- **Bans at every door.** A network-wide ban is enforced at the banned user's home server, so it does not recognise the same person arriving at another server. Recognising them everywhere would require an identifier to cross links: an account, or an address. A keyed hash does not help here, since the IPv4 space is small enough to search. Revisit if networks find evasion to be a real problem.
- **Server-scoped bans.** A ban that, like a kick, removes someone from one server only but persists across their sessions. The exclusion mechanism already carries it; it needs only a lifetime longer than a session.

## Implementation Notes

- **Allocations.** Capability bit 11, transactions 900–914 (915–919 reserved), fields `0x0630`–`0x063E` (`0x063F`–`0x064F` reserved) and reason codes 0–10 and 16–24 are assigned, as shipped in Janus 2.0.18. Interoperation has been exercised between Janus servers only.
- **Peer configuration.** The Janus reference design identifies a peer by a configuration entry. A dialing entry carries the peer's address and the credentials the peer issued; an accepting entry names the local account the peer logs in with. Features and the ghost bound are per entry. The server's own tag, its suggested color, the tag-display option and color overrides by tag are server-wide settings.
- **The ghost sink.** A reference server delivers transactions to users through a per-recipient outbox, and every broadcast loop (chat lines, user-list notifications) will reach ghosts through it. The outbox MUST **drop** every transaction addressed to a ghost. The translations above happen earlier: in the handlers for 108, 303 and 110, which recognise a ghost target before building any outbound transaction, and in the chat handler, which sends the raw text over links rather than forwarding the formatted 106 that the broadcast produced. A sink that drops everything is one choke point that is easy to keep closed. A sink that tried to translate whatever reached it would forward the formatted copy of every broadcast.
- **Hidden ghosts.** A user excluded at this server is still allocated an ID and kept in the per-link table so it can be relayed. It is a ghost with a "not listed" mark, handled by the same code that already keeps invisible users out of the list.
- **Failing closed on ghosts.** The reference server enforces [Transactions Naming a Ghost](#transactions-naming-a-ghost) in its transaction dispatcher, not in each handler: a request whose user ID names a ghost is refused before any handler runs, unless its type is one of the three that are translated (108, 303, 110). A handler added later is then safe by default, and the translated three are the only places that need to know about ghosts.
- **Joined, not connected.** The reference server marks a session as joined at the moment it announces the user to the user list (301), after the login reply and, where the server waits for one, after Agreed (121), and exports it from then on, as [Who Is Exported](#who-is-exported) requires. Until its login reply has been sent, a session is sent nothing at all, which also keeps chat from linked servers from reaching a client whose text encoding the server has not settled yet.
- **Operator state.** The server ID lives in its own file in the server's data directory (`server-id`), apart from the configuration an operator copies between servers, and `janus link reset-id` replaces it. Suspended links and trusted addresses are kept in `link-state.json`; bans a server enforces for others and bans it has placed on other servers' users are kept in separate files, so lifting one never touches the other.
- **Optional behaviour left out.** The reference server does not mark ghosts away during a grace period and does not post a chat notice when a link is lost. Both are MAYs.
