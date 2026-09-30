# Buddy Icons Extension

> **Status:** proposal. Nothing in this document is implemented by a server yet.

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document describes an addition to the [Instant Messaging Extension](Capabilities-Messaging.md): a small picture - a *buddy icon*, often an animated GIF - that an account publishes to its accepted friends, in the manner of the classic AIM buddy icon. The icon belongs to the **account**, not the session: it is stored by the server, survives sign-off, and is shown only to accepted friends. Clients learn that an icon changed from a hash carried with roster and presence data, and fetch the picture only when the hash differs from the one they have cached.

## Table of Contents

- [Background](#background)
  - [Why not the GIF Icons extension](#why-not-the-gif-icons-extension)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
  - [Server Configuration](#server-configuration)
  - [Server Limits Advertisement](#server-limits-advertisement)
- [Transaction Types](#transaction-types)
- [Data Objects](#data-objects)
- [Transactions](#transactions)
  - [Set Buddy Icon (827)](#set-buddy-icon-827)
  - [Get Buddy Icon (828)](#get-buddy-icon-828)
- [Changes to Existing Messaging Transactions](#changes-to-existing-messaging-transactions)
  - [Get Roster (800) and Roster Entry (801)](#get-roster-800-and-roster-entry-801)
  - [Presence Changed (809)](#presence-changed-809)
- [Storage](#storage)
- [Validation](#validation)
- [Privacy](#privacy)
- [Client Guidance](#client-guidance)

---

## Background

The messaging extension gives each account a Login, a display name, a status text and an optional profile ([User Profiles](Capabilities-Messaging.md)), but nowhere to put a picture. A buddy list without pictures is serviceable; one with them is what people remember from the messengers this extension is modelled on.

The requirements follow from the messaging layer's identity model:

- **Addressed by Login.** Friends know each other by Login, never by the 16-bit user ID of a session.
- **Account state.** Like the status text, the icon must be visible while its owner is offline, and must not have to be re-sent on every sign-on.
- **Friends only.** Like the profile, the icon is not published to strangers.
- **Cheap to keep current.** A buddy list may hold hundreds of entries; clients must be able to tell which icons changed without downloading any of them.

### Why not the GIF Icons extension

The [GIF Icons extension](GIF-Icons.md) (1861-1864) serves a different purpose and cannot be reused here:

- It is keyed by **User ID** (103), which is session-scoped and which the messaging layer deliberately never exposes for friends.
- Its Icon Change (1864) is broadcast to **every connected user**, which would leak an account's presence to people who are not its friends.
- Icons are stored **per session** and are not kept across reconnections.

The two extensions are independent. A server MAY implement both; a client MUST NOT assume an icon set through one is visible through the other.

---

## Compatibility and Negotiation

This extension is part of the messaging layer and is only available when `CAPABILITY_MESSAGING` (bit 6) is confirmed. It defines **no new capability bit**: a server signals support by advertising `DATA_MAX_ICON_BYTES` in the login reply (see below). A client that does not see that field MUST NOT send Set Buddy Icon (827) or Get Buddy Icon (828), and MUST NOT expect `DATA_BUDDY_ICON_HASH` in roster or presence data.

Everything is additive: a client that does not implement this extension ignores `DATA_BUDDY_ICON_HASH` as it ignores any unknown field, and sees no other change.

### Server Configuration

Using Janus as the reference, the settings would be:

| Setting | Type | Default | Description |
|---|---|---|---|
| `Messaging.BuddyIcons` | bool | `false` | Enable buddy icons |
| `Messaging.MaxIconBytes` | int | `16384` | Largest icon the server stores, in bytes |

### Server Limits Advertisement

When the server confirms `CAPABILITY_MESSAGING` **and** buddy icons are enabled, it MUST include in the login reply:

| Field ID | Name | Setting |
|---|---|---|
| `0x0623` | `DATA_MAX_ICON_BYTES` | `Messaging.MaxIconBytes` |

The rules of [Server Limits Advertisement](Capabilities-Messaging.md#server-limits-advertisement) apply: the server MUST NOT advertise more than it will accept, and a value of `0` is not meaningful (a client receiving it MUST use the default, 16384). Unlike the other limits, the field's **absence** is significant: it means the server does not support buddy icons.

16 KiB is deliberately small. The original AIM limit was 7 KB for a 48 x 48 image, and the animated icons people collected from that era rarely exceed a few kilobytes more.

---

## Transaction Types

Allocated from the range reserved for messaging growth (827-839):

| ID | Hex | Name | Direction |
|---:|---|---|---|
| 827 | `0x033B` | Set Buddy Icon | Client -> Server (request/reply) |
| 828 | `0x033C` | Get Buddy Icon | Client -> Server (request/reply) |

## Data Objects

Allocated from the fields reserved for messaging (`0x061D`-`0x061F`) and for server limits (`0x0623`-`0x0627`):

| Field ID | Dec | Name | Type | Description |
|---|---:|---|---|---|
| `0x061D` | 1565 | `DATA_BUDDY_ICON` | Binary | The picture: GIF (87a or 89a), PNG or JPEG bytes |
| `0x061E` | 1566 | `DATA_BUDDY_ICON_HASH` | Binary (16) | The icon's identity: the first 16 bytes of the SHA-256 of `DATA_BUDDY_ICON` |
| `0x0623` | 1571 | `DATA_MAX_ICON_BYTES` | UInt32 | Login reply only: the largest icon the server stores |

`DATA_BUDDY_ICON_HASH` identifies content, not a version: setting the same picture twice produces the same hash, and clients MAY share a cache between friends who use the same icon.

---

## Transactions

### Set Buddy Icon (827)

Request/reply. Sets or clears the caller's icon. The subject is always the caller's authenticated Login; there is no field naming whose icon to set.

**Request:**

| Field | Presence | Notes |
|---|---|---|
| `DATA_BUDDY_ICON` | REQUIRED | The picture; **empty** clears the icon |

**Reply (success):** `DATA_REASON_CODE` = `OK`, and `DATA_BUDDY_ICON_HASH` for the stored icon (absent when cleared).

**Reply (failure):** a non-zero error code with `FieldError` text, and `DATA_REASON_CODE`:

- `MessageTooLong` (13) - larger than `DATA_MAX_ICON_BYTES`.
- `RateLimited` (10) - the server limits how often an icon may change.
- No reason code (or a future dedicated one) for a picture that fails [validation](#validation); the `FieldError` text says why.

After a successful change the server MUST send **Presence Changed (809)** carrying the new `DATA_BUDDY_ICON_HASH` (empty when cleared) to every session of every accepted friend who is online, and to the caller's own other sessions. Friends who are offline learn the new hash from their next roster snapshot.

A server SHOULD NOT send 809 when the new hash equals the stored one.

### Get Buddy Icon (828)

Request/reply. Fetches one friend's icon.

**Request:** `DATA_FRIEND_LOGIN` (REQUIRED).

**Reply:**

| Case | Error code | Fields |
|---|---|---|
| Accepted friend (or self) with an icon | `0` | `DATA_FRIEND_LOGIN`, `DATA_BUDDY_ICON_HASH`, `DATA_BUDDY_ICON`, `DATA_REASON_CODE` = `OK` |
| Accepted friend (or self) without an icon | `0` | `DATA_FRIEND_LOGIN`, `DATA_REASON_CODE` = `OK` |
| Anyone else | `0` | `DATA_FRIEND_LOGIN`, `DATA_REASON_CODE` = `NotFriends` (6) |

As with Get User Info (825), `NotFriends` on an otherwise successful reply tells the client why nothing came back; it is not an error. A server MUST NOT distinguish "not a friend", "blocked you" and "no such account" in this reply.

A client SHOULD request an icon only when the hash it was given differs from the one it has cached, and SHOULD NOT poll.

---

## Changes to Existing Messaging Transactions

### Get Roster (800) and Roster Entry (801)

An `Accepted` entry whose friend has an icon carries `DATA_BUDDY_ICON_HASH`, **whether or not that friend is online** - the icon is account state, like `DATA_PRESENCE_STATUS_TEXT`, and the roster snapshot is where a client that has been away learns it. Entries in any other state MUST NOT carry it.

Because an 801 entry group is complete rather than incremental, an `Accepted` entry **without** `DATA_BUDDY_ICON_HASH` means the friend has no icon, and a client MUST drop any icon it was showing for them.

### Presence Changed (809)

When the icon changes, 809 carries `DATA_BUDDY_ICON_HASH` (empty when the icon was cleared). An 809 sent for another reason (a presence or status change) MAY omit the field; unlike the roster entry, 809 is a partial update, so **absence means unchanged** and only a present-but-empty field clears the icon.

---

## Storage

- The server stores at most one icon per account, with its hash, and keeps it until the owner replaces or clears it, or the account is deleted.
- The icon is stored as received (after validation). A server MAY re-encode it to strip metadata, in which case the hash MUST be computed over the stored bytes, not the uploaded ones.
- Storage cost is bounded by `MaxIconBytes` times the number of messaging accounts.

## Validation

- The server MUST check that `DATA_BUDDY_ICON` begins with a GIF87a, GIF89a, PNG or JPEG signature and MUST reject anything else.
- The server MUST reject an icon larger than `MaxIconBytes`.
- The server MAY reject icons whose decoded dimensions exceed a limit of its choosing (128 x 128 is generous for a 48 x 48 convention), and MAY limit how often an account changes its icon.

## Privacy

- Icons, and their hashes, are visible only to accepted friends and to the owner's own sessions.
- A blocked Login never receives the hash or the icon, and cannot tell a block from "no icon" or "no such account".
- Discovery (Find User 822, User Search 823) MUST NOT carry icons or hashes: finding someone is not being allowed to see their picture, the same rule as for profiles.

## Client Guidance

- Cache icons by hash, persistently; on sign-on, compare roster hashes with the cache and fetch only what changed.
- Draw icons at 48 x 48 (the classic size), scaling larger pictures down. Animated GIFs SHOULD animate; clients MAY stop animation after a while or on request.
- Decode defensively: a picture from another user is untrusted input. Cap decoded dimensions and frame count.
- A client that cannot render a format SHOULD show no icon rather than an error.
