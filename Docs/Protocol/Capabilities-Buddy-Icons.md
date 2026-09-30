# Buddy Icons Extension

> **Author:** John Leighow ([@tagban](https://github.com/tagban))

> **Status:** accepted (2026-09-30). Implemented by the reference server, Janus, from 2.0.16.

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
- [Reason Codes](#reason-codes)
- [Transactions](#transactions)
  - [Set Buddy Icon (827)](#set-buddy-icon-827)
  - [Get Buddy Icon (828)](#get-buddy-icon-828)
- [Changes to Existing Messaging Transactions](#changes-to-existing-messaging-transactions)
  - [Get Roster (800) and Roster Entry (801)](#get-roster-800-and-roster-entry-801)
  - [Presence Changed (809)](#presence-changed-809)
  - [Get User Info (825)](#get-user-info-825)
- [Changes to the Messaging Extension Document](#changes-to-the-messaging-extension-document)
- [Storage](#storage)
- [Validation](#validation)
- [Privacy](#privacy)
- [Client Guidance](#client-guidance)
- [Implementation Notes](#implementation-notes)

---

## Background

The messaging extension gives each account a Login, a display name, a status text and an optional profile ([User Profiles](Capabilities-Messaging.md#user-profiles)), but nowhere to put a picture. A buddy list without pictures is serviceable; one with them is what people remember from the messengers this extension is modelled on.

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

This extension is part of the messaging layer and is only available when `CAPABILITY_MESSAGING` (bit 6) is confirmed. It defines **no new capability bit**: a server signals support by advertising `DATA_MAX_ICON_BYTES` in the login reply (see below). A client that does not see that field MUST NOT send Set Buddy Icon (827) or Get Buddy Icon (828), and MUST NOT expect `DATA_BUDDY_ICON_HASH` in roster, presence or profile data.

Everything is additive: a client that does not implement this extension ignores `DATA_BUDDY_ICON_HASH` as it ignores any unknown field, and sees no other change.

### Server Configuration

Using Janus as the reference, the settings would be:

| Setting | Type | Default | Description |
|---|---|---|---|
| `Messaging.BuddyIcons` | bool | `false` | Enable buddy icons |
| `Messaging.MaxIconBytes` | int | `16384` | Largest icon the server stores, in bytes. At most `65535` |
| `Messaging.MaxIconDimension` | int | `128` | Largest width or height the server accepts, in pixels. At least `64` |
| `Messaging.MaxIconFrames` | int | `100` | Most frames the server accepts in an animated icon. At least `32`. Not advertised |

### Server Limits Advertisement

When the server confirms `CAPABILITY_MESSAGING` **and** buddy icons are enabled, it MUST include in the login reply:

| Field ID | Name | Setting |
|---|---|---|
| `0x0623` | `DATA_MAX_ICON_BYTES` | `Messaging.MaxIconBytes` |
| `0x0624` | `DATA_MAX_ICON_DIMENSION` | `Messaging.MaxIconDimension` |

The rules of [Server Limits Advertisement](Capabilities-Messaging.md#server-limits-advertisement) apply, with one exception:

- The server MUST NOT advertise more than it will accept, and a value of `0` is not meaningful. A client receiving `0` for `DATA_MAX_ICON_BYTES` MUST use the default, 16384.
- **Unlike the other limits, the absence of `DATA_MAX_ICON_BYTES` is significant: it means the server does not support buddy icons.** A client MUST NOT fall back to the default and assume support, as it would for the other limits. The messaging document states this exception too (see [Changes to the Messaging Extension Document](#changes-to-the-messaging-extension-document)), so a client reading only that document is not misled.

**The limit cannot exceed 65535.** A Hotline field's length is a UInt16, and the whole picture travels in one `DATA_BUDDY_ICON` field. The field is a UInt32 only for uniformity with the rest of the limits block. A server MUST NOT be configured, or advertise, above 65535, and a client receiving a larger value MUST treat it as 65535.

16 KiB is deliberately small. The original AIM limit was 7 KB for a 48 x 48 image, and the animated icons people collected from that era rarely exceed a few kilobytes more.

`DATA_MAX_ICON_DIMENSION` is the largest width or height, in pixels, that the server accepts, so a client can scale a picture to fit before uploading it instead of finding out from a refusal. **A client that does not receive it, or receives `0`, MUST assume 64** - the floor every server accepts (see [Validation](#validation)), and so the one value that is safe against any server. A client that always scales to the classic 48 x 48 need not read the field at all.

The frame limit is not advertised. It is covered by a floor instead: every server accepts at least 32 frames, which is more than most icons of the classic kind have, so a client with a longer animation trims it to 32 and is accepted everywhere. Keeping it out of the login reply leaves `0x0625`-`0x0627` free for limits that cannot be handled this way.

---

## Transaction Types

Allocated from the range reserved for messaging growth (827-839):

| ID | Hex | Name | Direction |
|---:|---|---|---|
| 827 | `0x033B` | Set Buddy Icon | Client -> Server (request/reply); Server -> Client (notification to the caller's other sessions) |
| 828 | `0x033C` | Get Buddy Icon | Client -> Server (request/reply) |

## Data Objects

Allocated from the fields reserved for messaging (`0x061D`-`0x061F`) and for server limits (`0x0623`-`0x0627`):

| Field ID | Dec | Name | Type | Description |
|---|---:|---|---|---|
| `0x061D` | 1565 | `DATA_BUDDY_ICON` | Binary (≤ 65535) | The picture: GIF (87a or 89a), PNG or JPEG bytes |
| `0x061E` | 1566 | `DATA_BUDDY_ICON_HASH` | Binary (16) | The icon's identity: the first 16 bytes of the SHA-256 of the stored `DATA_BUDDY_ICON` |
| `0x0623` | 1571 | `DATA_MAX_ICON_BYTES` | UInt32 | Login reply only: the largest icon the server stores (never above 65535) |
| `0x0624` | 1572 | `DATA_MAX_ICON_DIMENSION` | UInt32 | Login reply only: the largest width or height the server accepts, in pixels (never below 64) |

`DATA_BUDDY_ICON_HASH` identifies content, not a version: setting the same picture twice produces the same hash, and clients MAY share a cache between friends who use the same icon. It is never sent empty: an account without an icon is reported by leaving the field out.

## Reason Codes

This extension adds one value to `DATA_REASON_CODE` (`0x060F`) and widens the meaning of another:

| Code | Name | Meaning |
|---|---|---|
| 13 | `MessageTooLong` | *(widened)* Encoded body exceeds `MaxMessageBytes`, **or an icon exceeds `MaxIconBytes`** |
| 14 | `InvalidImage` | *(new)* The picture is not an accepted format, cannot be decoded, or exceeds the server's dimension or frame limits |

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
- `InvalidImage` (14) - fails [validation](#validation). The `FieldError` text says why.
- `RateLimited` (10) - the server limits how often an icon may change.

After a successful change that alters the stored hash, the server MUST send **Presence Changed (809)** to every session of every accepted friend who is online, **unless the caller is `Invisible`**. That 809 is the same complete notification the server sends for any other reason (see [Presence Changed (809)](#presence-changed-809)), so it carries the new hash, or omits it when the icon was cleared.

While the caller is `Invisible`, the server MUST NOT send it, for the reason given under [Privacy](#privacy). Friends pick up the new hash from their next roster snapshot, or from the 809 announcing that the caller has become visible again.

A server MUST NOT send 809 when the new hash equals the stored one.

#### Keeping the owner's other sessions in step

After a successful change that alters the stored hash, the server MUST also send **Set Buddy Icon (827) as a notification** to each of the caller's **other** sessions - every live session of the same Login except the one that made the change. It is sent whether or not the caller is `Invisible`: those sessions are the owner's own, not friends.

| Field | Presence | Notes |
|---|---|---|
| `DATA_BUDDY_ICON_HASH` | optional | The new hash; **absent** when the icon was cleared |

The notification uses task ID `0` with the *is-reply* flag unset, and carries **only the hash**, never the picture. A session receiving it compares the hash with its cache, and fetches its own icon with [Get Buddy Icon (828)](#get-buddy-icon-828) if they differ.

Reusing 827 in the server-to-client direction follows the precedent of the call-signalling transactions (814-821), which travel both ways under one number. Its direction tells the two apart: a client never receives an 827 request, and the server never receives an 827 notification. Clients that predate this extension ignore it, as they must any unrecognised transaction type.

809 is not used for this. It has only ever described a friend, and a client receiving one that names its own Login may add itself to its own buddy list.

The notification keeps sessions in step while they are signed on. A session that signs on later learns its own current hash from [Get User Info (825)](#get-user-info-825) on its own Login.

### Get Buddy Icon (828)

Request/reply. Fetches one friend's icon.

**Request:** `DATA_FRIEND_LOGIN` (REQUIRED).

**Reply:**

| Case | Error code | Fields |
|---|---|---|
| Accepted friend (or self) with an icon | `0` | `DATA_FRIEND_LOGIN`, `DATA_BUDDY_ICON_HASH`, `DATA_BUDDY_ICON`, `DATA_REASON_CODE` = `OK` |
| Accepted friend (or self) without an icon | `0` | `DATA_FRIEND_LOGIN`, `DATA_REASON_CODE` = `OK` |
| Anyone else | `0` | `DATA_FRIEND_LOGIN`, `DATA_REASON_CODE` = `NotFriends` (6) |
| Rate limit exceeded | non-zero | `FieldError`, `DATA_REASON_CODE` = `RateLimited` (10) |

As with Get User Info (825), `NotFriends` on an otherwise successful reply tells the client why nothing came back; it is not an error. A server MUST NOT distinguish "not a friend", "blocked you" and "no such account" in this reply.

The owner's presence plays no part: an icon is account state, so a friend who is offline or `Invisible` is answered exactly like one who is online.

A server MAY rate-limit 828 per account. If it does, the limit MUST allow a client to fetch every icon in a full roster within a few minutes of signing on. A client SHOULD request an icon only when the hash it was given differs from the one it has cached, and SHOULD NOT poll.

---

## Changes to Existing Messaging Transactions

The rule throughout is the one the messaging extension already uses for server-to-client data. **`DATA_BUDDY_ICON_HASH` is present whenever the friend has an icon, and its absence means the friend has none.** No transaction in this extension treats an absent hash as "unchanged". The messaging document explains, under [An entry group is complete, not incremental](Capabilities-Messaging.md#an-entry-group-is-complete-not-incremental), why that convention is not used in the server-to-client direction.

### Get Roster (800) and Roster Entry (801)

An `Accepted` entry whose friend has an icon carries `DATA_BUDDY_ICON_HASH`, **whether or not that friend is online or `Invisible`**. The icon is account state, like `DATA_PRESENCE_STATUS_TEXT`, and the roster snapshot is where a client that has been away learns it. Entries in any other state MUST NOT carry it.

Because an 801 entry group is complete rather than incremental, an `Accepted` entry **without** `DATA_BUDDY_ICON_HASH` means the friend has no icon, and a client MUST drop any icon it was showing for them.

### Presence Changed (809)

**Every** 809 carries `DATA_BUDDY_ICON_HASH` when the friend has an icon, whatever prompted it: a presence change, a status or name change, or an icon change. An 809 without the field means the friend has no icon, and a client MUST drop any icon it was showing for them, exactly as for a roster entry.

This costs 20 bytes per notification and removes any need for a per-field rule. It is also how the reference server already builds 809: from the account's current state, all at once, whenever anything changes. A server SHOULD build the hash into that same routine rather than adding it only to icon-change notifications.

An 809 sent for an icon change is a complete presence notification like any other. It carries the friend's current `DATA_PRESENCE_STATE` (REQUIRED, as always) along with the other fields the server holds.

### Get User Info (825)

When the caller is an accepted friend of the subject, or is the subject, the reply carries `DATA_BUDDY_ICON_HASH` if the subject has an icon, alongside the `DATA_PROFILE_*` fields. The non-friend card (`NotFriends`) MUST NOT carry it.

This is also how a client learns **its own** icon. After signing on, a client asks 825 about its own Login and compares the hash with its cache. It needs to download its own icon only when the hash differs, and it does not have to download it just to find out whether one is set.

---

## Changes to the Messaging Extension Document

Adopting this extension requires these edits to [Capabilities-Messaging.md](Capabilities-Messaging.md), so that neither document contradicts the other:

1. **Field table** - move `0x061D`, `0x061E`, `0x0623` and `0x0624` from "reserved" to allocated, pointing here, and narrow the reserved ranges to `0x061F` and `0x0625`-`0x0627`.
2. **Transaction table** - list 827 and 828, and narrow the reserved range to 829-839. Under Transaction Semantics, add both to the request/reply list, and add 827 to the server-initiated notification list (sent to the caller's other sessions).
3. **Server Limits Advertisement** - after "Clients MUST tolerate any of these fields being absent and fall back to the defaults", add: *"The exceptions are the fields defined by the [Buddy Icons extension](Capabilities-Buddy-Icons.md): the absence of `DATA_MAX_ICON_BYTES` (`0x0623`) means the server does not support buddy icons, and an absent `DATA_MAX_ICON_DIMENSION` (`0x0624`) means 64."*
4. **Reason Codes** - widen `MessageTooLong` (13) and add `InvalidImage` (14), as in [Reason Codes](#reason-codes).
5. **Get Roster (800), Roster Entry (801), Presence Changed (809), Get User Info (825)** - list `DATA_BUDDY_ICON_HASH` (optional) among the fields, with a link here.
6. **Persistence** - add the buddy icon to the per-account state a server SHOULD persist.

---

## Storage

- The server stores at most one icon per account, with its hash, and keeps it until the owner replaces or clears it.
- The icon is stored as received, after validation. A server MAY re-encode it to strip metadata. If it does, the hash MUST be computed over the stored bytes, not the uploaded ones.
- Storage cost is bounded by `MaxIconBytes` times the number of messaging accounts.
- The icon follows the account through the messaging extension's [account-lifecycle events](Capabilities-Messaging.md#persistence):
  - **Deletion** - the icon is deleted with the account.
  - **Rename** - the icon moves to the new Login. The fresh roster entry that friends receive for the new Login carries its hash.
  - **Revocation of `AccessMessaging`** - treated as deletion, so the icon is deleted too. It would otherwise outlive the messaging identity it belongs to.

## Validation

A signature check alone is not enough. A few bytes of header can declare an image of 65535 x 65535, and every friend who decodes it pays for that. The server therefore reads the image's own header:

- The server MUST check that `DATA_BUDDY_ICON` begins with a GIF87a, GIF89a, PNG or JPEG signature and MUST reject anything else.
- The server MUST reject an icon larger than `MaxIconBytes`.
- The server MUST read the declared dimensions from the image header (the GIF logical screen descriptor, the PNG `IHDR` chunk, or the JPEG `SOFn` marker). It MUST reject an image whose width or height is larger than `MaxIconDimension`, or whose header is truncated or unreadable.
- `MaxIconDimension` MUST NOT be set below 64, so that a client assuming the floor is accepted by every server.
- The server SHOULD cap the frame count of an animated GIF or APNG at `MaxIconFrames`, and MUST accept at least 32 frames.
- The server MAY limit how often an account changes its icon.

All of these failures except the size limit are reported as `InvalidImage` (14). A server that already validates images for [Inline Media](Capabilities-Inline-Media.md) MAY reuse that code path, provided the result still fits the limits above.

## Privacy

- Icons, and their hashes, are visible only to accepted friends and to the owner's own sessions.
- A blocked Login never receives the hash or the icon, and cannot tell a block from "no icon" or "no such account".
- Discovery (Find User 822, User Search 823) MUST NOT carry icons or hashes: finding someone is not being allowed to see their picture, the same rule as for profiles.
- **Changing an icon while `Invisible` MUST NOT reach friends until the owner becomes visible.** Friends see an `Invisible` account as signed out. An 809 from a friend who is signed out is proof that the friend is not, so sending one would defeat `Invisible` entirely. The roster snapshot and 828 still serve the new icon, because a stored icon, unlike a notification, says nothing about whether its owner is connected.

## Client Guidance

- Cache icons by hash, persistently. On sign-on, compare the roster hashes with the cache and fetch only what changed.
- **Fetch lazily.** Request icons for the rows on screen first, one or a few at a time, rather than every stale icon at once. A full roster of changed icons can run to several megabytes, and a server may rate-limit 828. On `RateLimited`, back off and retry later rather than giving up on the icon.
- Scale icons to fit **before uploading**: 48 x 48 (the classic size) is RECOMMENDED, and never more than `DATA_MAX_ICON_DIMENSION` (64 when the server does not send it). Trim animations longer than 32 frames unless you are prepared to handle `InvalidImage`.
- Handle an 827 notification by comparing its hash with your own cached icon, and fetch with 828 on a mismatch; an 827 without a hash means your icon was cleared from another session.
- Draw icons at 48 x 48 as well, scaling larger pictures down. Animated GIFs SHOULD animate; clients MAY stop animation after a while or on request.
- Decode defensively: a picture from another user is untrusted input. Cap decoded dimensions and frame count even though the server has checked them.
- A client that cannot render a format SHOULD show no icon rather than an error.

## Implementation Notes

This section is informative. It records how the reference server, Janus, meets this document where the document leaves the choice to the server.

- **Configuration.** The settings are as in [Server Configuration](#server-configuration). Values outside the protocol's bounds are corrected rather than refused: `MaxIconBytes` above 65535 is treated as 65535, and `MaxIconDimension` and `MaxIconFrames` below their floors are raised to 64 and 32.
- **Validation** reuses the server's [Inline Media](Capabilities-Inline-Media.md) validator. It also rejects polyglot files with data after the image's end marker, probes dimensions before allocating pixel memory, and bounds decoding time.
- **Re-encoding.** Janus stores the validator's re-encoded image, which removes metadata such as a phone photo's EXIF location. A re-encode can come out larger than the upload. When it no longer fits `MaxIconBytes` but the upload did, Janus stores the upload instead, with its metadata removed losslessly:
  - **PNG:** only the chunks needed to draw the image are kept (`IHDR`, `PLTE`, `IDAT`, `IEND`, `tRNS`, `gAMA`, `cHRM`, `sRGB`).
  - **JPEG:** `APPn` and comment segments are removed, except `APP0` (JFIF) and `APP14` (Adobe), which affect how colour is decoded.
  - **GIF:** stored as uploaded.

  An upload within the advertised limit is therefore never refused with `MessageTooLong`.
- **Animated PNG** is stored as its default image. Neither path keeps the APNG animation chunks, so the icon is static. Animated GIFs keep their animation, and their frame count is checked against `MaxIconFrames`.
- **Rate limits** are per account:
  - 827 allows a burst of 3, then one change every five seconds.
  - 828 allows a burst of 50, then 5 per second, so a full default roster of 500 changed icons downloads in about 90 seconds.
- **Storage.** Icons are kept in their own table in the messaging database. It was added without a schema version change, so an existing database gains it on the next start.
- **Switching the feature off** keeps stored icons but stops serving them: the limits are no longer advertised, `DATA_BUDDY_ICON_HASH` is left out of roster, presence and profile data, and 827 and 828 are refused with `FieldError` text and no reason code.
- **Task ID.** The 827 echo, like every messaging notification Janus sends, carries task ID `0`.
