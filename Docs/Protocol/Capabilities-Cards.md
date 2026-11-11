# Card Store Extension

> Last updated: November 11, 2026

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This extension lets a server keep **cards** and **libraries** in a store of its own, and answer a client's queries: one page of items, with filter counts, for a tab, ticks, a search and a sort. A client that would otherwise walk a folder of `.card` files asks once and gets the page it shows.

How a card is read, and everything a library derives from it, is defined in [Card Format 1.0 – Reading Rules](Cards-Format.md). This document defines only the transactions. For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md).

## Table of Contents

- [Motivation](#motivation)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
- [The Model](#the-model)
- [Fields](#fields)
- [JSON on the Wire](#json-on-the-wire)
- [Transactions](#transactions)
- [List Libraries (1000)](#list-libraries-1000)
- [Query Library (1001)](#query-library-1001)
- [Get Card (1002)](#get-card-1002)
- [Writing Cards (1003–1005)](#writing-cards-10031005)
- [Writing Libraries (1006–1008)](#writing-libraries-10061008)
- [Comments and Ratings (1009–1015)](#comments-and-ratings-10091015)
- [Hash Files (1016)](#hash-files-1016)
- [Errors](#errors)
- [Privileges](#privileges)
- [Visibility](#visibility)
- [Server Behaviour](#server-behaviour)
- [Client Behaviour](#client-behaviour)
- [Security Considerations](#security-considerations)
- [Allocations](#allocations)
- [Example](#example)

---

## Motivation

A client can show a library on any server by walking it: listing the folders, downloading every `.card` file, reading them and picking out items itself. That costs a listing per folder and a download per card, on every open, and a client has to stop somewhere. Zephyr stops at 2,000 files.

A server with thousands of items, such as an archive with a card for each of 6,000 applications, is past what a walk can do. Walking it would take minutes, and the client would still have to hold every card to answer a search.

A store answers in one request whatever its size, because the server indexes the cards when they are written. It also gives cards a home that isn't a file anyone can overwrite. A curator manages cards and libraries under privileges of their own, and the server checks every card when it is written, instead of a client finding a broken one on open.

## Compatibility and Negotiation

- A new capability bit, `CAPABILITY_CARDS` (bit 16, mask `0x00010000`), in `DATA_CAPABILITIES`. A client advertises it at login; a server with a card store confirms it. Bit 16 does not fit in a 2-byte field: a client that sets it sends the field 4 bytes wide, and a server that confirms it replies at that width ([Field Width](Capabilities.md#field-width)).
- A server MUST NOT confirm bit 16 unless it also confirms `CAPABILITY_TEXT_ENCODING` (bit 1). Cards, and every name in them, are UTF-8.
- A server MUST NOT confirm bit 16 on a server link (bit 11).
- A client MUST NOT send the transactions of this extension unless the server confirmed the bit. Without it, the client walks.
- A server that confirms the bit still serves its files as before. A `.card` file in its file area is an ordinary file: a client may View it and walk it, and the server doesn't read it as a card unless an operator imports it.

## The Model

**A card** is a stored package card ([Package Cards](Cards-Format.md#package-cards)) with:

| | |
|---|---|
| **Key** | The server's name for it: an opaque string of 1–64 ASCII characters, stable for the card's life, never reused. The key alone tells cards apart: the store may hold any number of cards with the same `id` or name ([Nothing a card says is unique](Cards-Format.md#the-document)) |
| **Home** | A folder in the file area. The card's paths resolve from there ([Paths](Cards-Format.md#paths)) |
| **File name** | The name it was stored with, used only as the display name of a card without a `name`. Two cards in one home folder can't have the same file name, as two files can't; a card may have none |
| **Revision** | A whole number, raised by every change to the card |
| **Document** | Its JSON text, kept and returned **exactly as written** |

**A library** is a stored index card ([Index Cards](Cards-Format.md#index-cards)) with a key, a revision and a **scope**: a folder in the file area. The library holds every card whose home is the scope or a folder inside it, at any depth.

A card's files are **ordinary files** in the file area. Pictures, descriptions, builds, tracks and books are fetched, previewed and downloaded by the usual transactions. The store records nothing about them but what the card says; whether each exists and its size come from the file area when a reply is made.

**The sequence number** is a whole number the server raises whenever a change may alter a reply: a card or library written, or a file one of them names added, removed or changed. A reply carries it, so a client can tell when what it holds is out of date.

## Fields

| ID | Decimal | Name | Type |
|---|---|---|---|
| `0x0400` | 1024 | `DATA_CARD_JSON` | Binary: part of a JSON document. May repeat |
| `0x0401` | 1025 | `DATA_CARD_ERROR` | Unsigned integer, big-endian, 2 bytes: a reason code ([Errors](#errors)) |

Field IDs `0x0402`–`0x041F` are reserved for this extension's growth.

## JSON on the Wire

Every request and reply of this extension carries one JSON document ([RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259)), in UTF-8, as the data of `DATA_CARD_JSON`.

- **Chunks.** A field holds at most 65,535 bytes. A longer document is split into consecutive `DATA_CARD_JSON` fields. The receiver joins them in the order they appear, then decodes the whole. A split may fall anywhere, inside a character included. A sender SHOULD fill every field but the last.
- **Never transcoded.** The field is binary. The text encoding extension does not apply to it, and a server MUST NOT convert it.
- **Size.** A request's document MUST NOT exceed 262,144 bytes (256 KB), and a reply's 786,432 bytes (768 KB). A client MUST accept a transaction of up to 1 MiB.
- **Names and paths.** A folder is a JSON array of its names from the root of the session's file area, each a string, as Get File Name List (200) names them. `[]` is the root.
- **Members** this document doesn't define MUST be ignored by a receiver. A request member of the wrong type is an [invalid request](#errors).
- **Cards inside documents.** Where a request or reply holds a card's or index card's document, it holds it as a JSON value, not as a string. A server MUST return a stored document's text exactly as it was written, inside the reply.

## Transactions

| ID | Hex | Name | Sent by | Needs |
|---|---|---|---|---|
| 1000 | `0x03E8` | List Libraries | Client | Download |
| 1001 | `0x03E9` | Query Library | Client | Download |
| 1002 | `0x03EA` | Get Card | Client | Download |
| 1003 | `0x03EB` | Add Card | Client | Manage cards (64) |
| 1004 | `0x03EC` | Change Card | Client | Manage cards (64) |
| 1005 | `0x03ED` | Delete Card | Client | Manage cards (64) |
| 1006 | `0x03EE` | Add Library | Client | Manage libraries (65) |
| 1007 | `0x03EF` | Change Library | Client | Manage libraries (65) |
| 1008 | `0x03F0` | Delete Library | Client | Manage libraries (65) |
| 1009 | `0x03F1` | Get Comments | Client | Download |
| 1010 | `0x03F2` | Post Comment | Client | Comment (66) |
| 1011 | `0x03F3` | Edit Comment | Client | Comment (66), own comments |
| 1012 | `0x03F4` | Delete Comment | Client | Comment (66) for its own; Moderate (68) for any |
| 1013 | `0x03F5` | Hide Comment | Client | Moderate (68) |
| 1014 | `0x03F6` | Rate Card | Client | Rate (67) |
| 1015 | `0x03F7` | Clear Ratings | Client | Moderate (68) |
| 1016 | `0x03F8` | Hash Files | Client | Manage cards (64) |

Transaction IDs 1017–1019 (`0x03F9`–`0x03FB`) are reserved for this extension's growth.

Every transaction is a request with a reply. A failure is an ordinary error reply, as described in [Errors](#errors).

### Items

A reply describes each card it returns as an **item**, a JSON object (in Query Library's `products` array, or as `product`):

```json
{
  "key": "c17",
  "revision": 3,
  "home": ["Software", "PixelCraft"],
  "file": "pixelcraft.card",
  "files": {"pixelcraft-4.2.0-mac.zip": 218, "pixelcraft-4.2.0-win.zip": 218, "thumb.png": 3391},
  "card": { "card_version": "1.0", "kind": "package", "type": "application", "name": "PixelCraft" }
}
```

| Member | |
|---|---|
| `key`, `revision`, `home`, `file` | As in [The Model](#the-model) |
| `files` | Every path the card names that resolves and exists as a file, with its size. A path is given as its resolved names joined by `/`. A path absent from `files` is refused or missing |
| `card` | The document, exactly as written |
| `rating` | The card's ratings: `{"stars": [n1, n2, n3, n4, n5]}`, how many accounts gave one to five stars. Absent when it has none |
| `comments` | How many comments the session can read. Absent when none |

The paths in `files` include those of every version, track, episode, chapter, screenshot, thumbnail and description, so a client needs nothing more to show the card.

`rating` and `comments` are as they stand when the reply is made. Changes to them don't raise the sequence number: they don't move an item between pages, and a busy library would otherwise make every cursor stale.

## List Libraries (1000)

**Request:** no fields. A `DATA_CARD_JSON` of `{}` is the same.

**Reply:**

```json
{
  "seq": 812,
  "libraries": [
    {"key": "s1", "revision": 1, "scope": ["Software"], "cards": 332,
     "index": {"card_version": "1.0", "kind": "index", "title": "Software & Games"}}
  ]
}
```

These are the libraries the session [can see](#visibility), in the order of their scopes: name by name, by code point, a shorter path first. `cards` counts the cards in each that the session can see. `index` is the index card, exactly as written.

A server with no library the session can see replies with an empty list.

## Query Library (1001)

**Request:**

```json
{
  "library": "s1",
  "tab": 1,
  "ticked": {"platform": ["linux", "classic"], "channel": ["beta"]},
  "search": "tide left",
  "sort": "newest",
  "limit": 50,
  "cursor": null
}
```

| Member | | Absent |
|---|---|---|
| `library` | A library's key | Required |
| `tab` | The tab, by its place (from 0) among the library's tabs, or its [default tabs](Cards-Format.md#default-tabs) | `0` |
| `ticked` | Facet keys, each with its ticked values | Nothing ticked |
| `search` | The search, as typed | No search |
| `sort` | `newest`, `name` or `size` | The index card's `sort` |
| `limit` | The most items to return, 1–200 | `50` |
| `cursor` | Where a previous reply left off | The first page |

A tab past the last, a facet key the [reading rules](Cards-Format.md#facets) don't define, or a sort other than the three is an [invalid request](#errors).

**Reply:**

```json
{
  "seq": 812,
  "total": 4,
  "tabs": [
    {"label": "All", "type": null, "count": 4},
    {"label": "Games", "type": "game", "count": 1},
    {"label": "Applications", "type": "application", "count": 3}
  ],
  "facets": [
    {"key": "platform", "values": [{"value": "linux", "count": 3}, {"value": "classic", "count": 1}]}
  ],
  "products": [ ],
  "next": "opaque"
}
```

| Member | |
|---|---|
| `seq` | The sequence number the reply was made at |
| `total` | How many items pass, over all pages |
| `tabs` | The library's tabs, or its default tabs, with each tab's [count](Cards-Format.md#counts). `type` is a kind, or `null` for a tab of every kind. First page only |
| `facets` | The library's facets, in its order, each with its [counts](Cards-Format.md#counts) in their order. A facet with no values is left out. First page only |
| `products` | The page's [items](#items), in the sort's [order](Cards-Format.md#order) |
| `next` | A cursor for the next page, or `null` when this is the last |

The tabs, counts, items and order are those of the [reading rules](Cards-Format.md#picking-items), over the cards in the library that the session can see. The default tabs, too, come from the cards the session can see.

**Pages.** A page holds at most `limit` items. A server MAY stop a page sooner to keep the reply within its size, but MUST return at least one item while any remain. A cursor is an opaque string of at most 256 bytes. It is valid only with the request it came from, with `cursor` changed. A server MAY refuse a cursor once the sequence number has moved on ([Stale](#errors)), and the client then starts again from the first page.

## Get Card (1002)

**Request:** `{"key": "c17"}`, a card by its key; or `{"home": ["Software", "Transit"], "file": "transit.card"}`, the card stored at that place. Not both. `home` is the card's home folder, as names from the root of the session's file area, and `file` the file name it was stored with, matched exactly. A client asks by place to open a card another card names ([Related Cards](Cards-Format.md#related-cards)).

**Reply:** `{"seq": 812, "product": { ... }}`, the card as an [item](#items).

A place where no card is stored, and one the session can't see, are answered as an unknown key is: **NotFound**. A client that asked by place then reads the `.card` file there, if any.

## Writing Cards (1003–1005)

**Add Card (1003).** `{"home": [...], "file": "pixelcraft.card", "card": { ... }}`. `file` is optional. The reply is `{"seq": ..., "product": { ... }}`, the card as stored, with its new key.

**Change Card (1004).** `{"key": "c17", "revision": 3, "home": [...], "file": "...", "card": { ... }}`. `home`, `file` and `card` are each optional, and what is absent is kept. The reply is as for Add Card.

**Delete Card (1005).** `{"key": "c17", "revision": 3}`. The reply is `{"seq": ...}`. **Deleting a card never deletes a file.**

A server MUST refuse a write, storing nothing, when:

- **the card doesn't read as a package card** ([The Document](Cards-Format.md#the-document)): it is malformed, not a card, too complex, or an index card. The reply carries the [problem](#problems);
- **the home folder doesn't exist**, or the session can't see it ([Not Found](#errors));
- **another card in the home folder has its file name** ([Exists](#errors));
- **the card has neither a `name` nor a file name** ([Invalid](#errors));
- **`revision` is not the card's current one.** Someone changed it since the client read it ([Changed](#errors)).

A server MUST NOT refuse a card for anything else these rules read as absent: a missing file, a refused path or a value of the wrong type. A card a walk would show, a store accepts. Nor may it refuse or replace a card because another has the same `id`, name or anything else the card says: those are different items.

### Problems

A refusal because the card doesn't read carries a reply document:

```json
{"problem": {"kind": "malformed", "line": 3, "column": 1, "message": "EOF while parsing an object"}}
```

`kind` is `malformed` (with `line`, `column` and `message`), `not_a_card`, `too_complex` (with `message`) or `index_card`. Lines and columns count from 1, and columns count characters. A message is for a person; its wording is not specified.

## Writing Libraries (1006–1008)

**Add Library (1006).** `{"scope": [...], "index": { ... }}`. The reply is `{"seq": ..., "library": { ... }}`, as an element of List Libraries.

**Change Library (1007).** `{"key": "s1", "revision": 1, "scope": [...], "index": { ... }}`. `scope` and `index` are each optional.

**Delete Library (1008).** `{"key": "s1", "revision": 1}`. **Deleting a library never deletes a card.**

A server MUST refuse, storing nothing:

- an `index` that doesn't read as an index card, with the [problem](#problems), or with kind `package_card` when it is a package card;
- a scope that doesn't exist, or that the session can't see;
- a scope that another library already has ([Exists](#errors));
- a `revision` that isn't the library's current one ([Changed](#errors)).

## Comments and Ratings (1009–1015)

Accounts may comment on a card and rate it, under [privileges](#privileges) of their own. A session reads the comments and ratings of every card it can see.

### Comments

A **comment** is:

| Member | |
|---|---|
| `id` | The server's name for it: an opaque string of 1–64 ASCII characters, never reused |
| `name` | The name its author used when posting |
| `posted` | When it was posted: UTC, as RFC 3339 (`2026-10-30T14:05:00Z`) |
| `edited` | When it was last edited, as `posted`. Absent if never |
| `text` | Plain text, with lines separated by LF |
| `mine` | `true` when the session's account posted it. Absent otherwise |
| `hidden` | `true` when a moderator hid it. Sent to moderators only |
| `account` | The account that posted it. Sent to moderators only |

**Text** is trimmed of leading and trailing white space, has CR LF and lone CR turned into LF, and MUST then be 1–4,096 bytes of UTF-8. A comment is text, never markup: a client shows it as written.

**Get Comments (1009).** `{"key": "c17", "limit": 50, "cursor": null}`. `limit` is 1–200, 50 when absent. The reply:

```json
{"total": 2, "rating": {"stars": [0, 0, 1, 3, 2]}, "mine": 4,
 "comments": [{"id": "m9", "name": "Ada", "posted": "2026-10-30T14:05:00Z", "text": "Runs well on a G4.", "mine": true}],
 "next": "opaque"}
```

Comments come newest first. `total` counts those the session can read. `rating` is as in an [item](#items), absent with no ratings, and `mine` is the stars the session's account gave, absent if none. A cursor is valid with the same `key` and `limit`, and stays valid while comments are posted: the next page starts after the last comment of the one before.

A hidden comment is read only by moderators. To everyone else it is gone, from the list and from every count.

**Post Comment (1010).** `{"key": "c17", "text": "..."}`. The reply is `{"comment": { ... }}`. The name is the session's user name.

**Edit Comment (1011).** `{"id": "m9", "text": "..."}`. Only the account that posted a comment may edit it. The reply is as for Post Comment.

**Delete Comment (1012).** `{"id": "m9"}`. Its author's account deletes it with bit 66, any other with bit 68. The reply is `{}`.

**Hide Comment (1013).** `{"id": "m9", "hidden": true}`, or `false` to show it again. The reply is `{"comment": { ... }}`.

A comment on a card the session can't see, and a hidden comment asked about by anyone but a moderator, is [Not Found](#errors). A comment the session can read but may not edit, delete or hide is [Denied](#errors).

### Ratings

A **rating** is one to five stars, from one account for one card.

**Rate Card (1014).** `{"key": "c17", "stars": 4}`. `stars` is 1–5 to rate, replacing the account's earlier rating, or 0 to withdraw it. The reply is `{"rating": { ... }, "mine": 4}`, as in Get Comments.

**Clear Ratings (1015).** `{"key": "c17"}` clears every rating of the card; `{"key": "c17", "account": "guest"}` only that account's. The reply is as for Rate Card, with `mine` the moderator's own (absent after a full clear).

A card that is deleted takes its comments and ratings with it. A card that is changed, or moved, keeps them.

## Hash Files (1016)

A curator writing a card names its files and, for each, the SHA-256 a client checks a download against. Hash Files has the server work them out where the files are, so the curator need not download them.

**Hash Files (1016).** `{"home": ["Software", "PixelCraft"], "files": ["files/pixelcraft-4.2.0-mac.dmg", "cover.png"]}`. `home` is a folder, and each of `files` is a [path](Cards-Format.md#paths) resolved inside it, as a card's paths are. `files` holds 1–64 paths. The reply:

```json
{"files": [{"path": "files/pixelcraft-4.2.0-mac.dmg", "size": 52428800, "sha256": "9f86d0..."},
           {"path": "cover.png", "missing": true}]}
```

There is one element for each path asked, in the order asked, with `path` as written. A file has `size`, as Get File Name List (200) reports it, and `sha256`: 64 lowercase hex digits. A path that is refused, names a folder, or names nothing the session can see has `"missing": true` instead. A file larger than the server will hash has its `size` and `"too_large": true`, and no `sha256`.

- **What is hashed** is the file's data fork: the bytes a client saves, and checks against the card. A file's resource fork, and its Finder information, are not hashed. An alias is hashed as its target is.
- **Visibility** is that of a card in `home`. A home the session can't see is [Not Found](#errors), and a file in it the session can't see is missing.
- **Hashing costs the server.** A server MAY decline Hash Files altogether, replying Denied to every session, or set a size past which it doesn't hash a file. It MAY take one Hash Files request of a session at a time, replying Busy to another meanwhile. It SHOULD keep what it hashed, by the file and its size and modification date, and answer from that until the file changes. A client asking about large files SHOULD ask about a few at a time, and expect a reply to take as long as reading them does.

Hash Files writes nothing: a card says what its author wrote, and the server still does not check it ([Checksums](#security-considerations)).

## Errors

An error reply has a non-zero error code and a message for the user in `DATA_ERROR` (100), as every Hotline error reply does. It also carries a reason in `DATA_CARD_ERROR`:

| Code | Name | When |
|---|---|---|
| 1 | Invalid | The request breaks this document: JSON that doesn't decode, a member of the wrong type, a value out of range |
| 2 | Not Found | No library, card or folder by that key or path that the session can see |
| 3 | Denied | The session lacks the privilege |
| 4 | Refused | The card or index card doesn't read. A `DATA_CARD_JSON` carries the [problem](#problems) |
| 5 | Exists | Another card in the home folder has the file name, or another library has the scope |
| 6 | Changed | The `revision` given isn't the current one |
| 7 | Stale | The cursor is no longer valid |
| 8 | Busy | The server is limiting the session's requests. Try again later |
| 9 | Too Large | The request, or the reply it would need, is past the size limits |

A server MUST NOT tell **Not Found** from **Denied** for something the session can't see. That something exists is itself what [Visibility](#visibility) keeps from it.

## Privileges

| Bit | Name | Grants |
|---|---|---|
| 2 | `AccessDownloadFile` | List Libraries, Query Library and Get Card. A card is viewed as a file is |
| 64 | `AccessManageCards` | Add, Change and Delete Card, and Hash Files |
| 65 | `AccessManageLibraries` | Add, Change and Delete Library |
| 66 | `AccessCommentCards` | Post Comment, and Edit and Delete Comment of its own account's comments |
| 67 | `AccessRateCards` | Rate Card: one rating per account per card, which it may change |
| 68 | `AccessModerateCards` | Hide Comment, Delete Comment of anyone's, and Clear Ratings. A moderator reads hidden comments and every comment's account |

Bits 64 to 68 are in the [extended privilege bitmap](Capabilities-Extended-Priv.md).

- **Granting them.** Only a path that can represent a bit past 63 can grant these bits: the server's own administration, or a client with `CAPABILITY_EXTENDED_PRIV` (bit 5) confirmed. As that extension requires, an account editor that can't see them preserves them.
- **Checking them.** A server checks the account's bits whether or not the session negotiated bit 5. A client learns that it holds them from the extended bitmap, and SHOULD offer management only then.
- **They are grants.** A clear bit is what every account had before this extension.

Holding bits 64 to 68 does not widen [visibility](#visibility). A curator manages, and a moderator moderates, only what they can see.

**Comments and ratings:**

- **Reading them needs no bit of its own.** A session that can see a card can read its comments and its rating, so a comment never reveals a card the session can't see.
- **Commenting and rating are separate bits.** An operator may take ratings from every account but comments only from trusted ones. Moderation is a third bit, so a moderator need not be able to edit cards, nor a curator to moderate.

Ratings are kept per account, not per session: a guest account shared by many people has one rating per card.

## Visibility

A session **can see a folder** when Get File Name List (200) of that folder would succeed for it, and every folder on the way is listed in its parent. A folder the server hides, a drop box the session may not view, and a folder outside an account's own file root can't be seen.

- A session sees a **card** when it can see the card's home folder.
- A session sees a **library** when it can see its scope.
- Every reply counts and returns only what the session can see.

A card whose home folder no longer exists is seen by nobody until it is changed to another home or the folder returns.

## Server Behaviour

- **Read speed over write speed.** A store SHOULD index what queries need when a card is written: the facet values, the search words, the date, and the item size once its files are known. A query then answers from the indexes. An index MAY narrow candidates in its own way, but the result MUST be that of the reading rules.
- **File changes.** A file a card names may be uploaded, replaced, renamed or deleted through the server at any time. A store SHOULD follow those changes (raising the sequence number), so that sizes, item sizes and the `size` order stay right. It MAY follow a move of a card's home folder made through the server.
- **Import.** A server SHOULD offer its operator a way to import the `.card` files in a folder. Each card is stored with its own folder as home and its file name, and each `index.card` becomes a library scoped to its folder. Each card is checked as a write would be. Importing again SHOULD replace the card with the same home and file name, and nothing else: two cards with the same `id` or name in different places are two items. An import is not a transaction of this extension.
- **Rate limits.** A server MAY limit how often a session queries, replying with Busy.
- **Logging.** A refused write SHOULD be logged with its problem and the account.

## Client Behaviour

- **Ask, don't walk.** With the bit confirmed, a client SHOULD show a server's libraries from List Libraries, and their items from Query Library. It SHOULD fetch a page as it is about to show it, using `total` to size its scroll bar.
- **Keep one reader.** An item's `card` is read by the same rules as a walked card. Sizes and existence come from its `files`.
- **The sequence number.** A client MAY keep pages and cards it has, and SHOULD reload them when a reply carries a different sequence number.
- **Pictures and files** are fetched as files, from the item's home folder. Thumbnails are fetched as items are shown, not for every item.
- **A `.card` file Viewed directly** is read from the file, as on any server. It is an ordinary file, and it may not be in the store.

## Security Considerations

- **A card is data, never instructions.** Nothing in a card chooses what the server reads, runs or serves. Its paths are resolved by the reading rules, inside its home folder. A path that leaves the home folder is refused, and a file is served only by the transactions that serve any file, with their privileges.
- **Sizes and existence are the server's,** never the card's. A card can't make a missing file look present or a large one look small.
- **Checksums** are the author's claim. A client verifies a downloaded file against them; the server does not, though it may work them out for a curator ([Hash Files](#hash-files-1016)).
- **Queries cost the server.** The page size, the request and reply limits, and the rate limit bound a session's load. A server SHOULD also bound the time it spends on one query.
- **Visibility is enforced on every reply,** counts included: a facet count must not reveal a card the session can't see.
- **Comments name people.** A comment carries the name its author chose, never the account, except to moderators. A server SHOULD rate-limit posting as it does chat.

## Allocations

| What | Allocated | Reserved |
|---|---|---|
| Capability bit | 16, `CAPABILITY_CARDS` | |
| Transactions | 1000–1016 | 1017–1019 |
| Fields | `0x0400`, `0x0401` | `0x0402`–`0x041F` |
| Privilege bits | 64 `AccessManageCards`, 65 `AccessManageLibraries`, 66 `AccessCommentCards`, 67 `AccessRateCards`, 68 `AccessModerateCards` | |

## Example

A client asks for the libraries. The transaction header is as usual; the fields follow.

```
Request:  type 1000 (0x03E8), no fields.

Reply:    00 01                         one field
          04 00  00 8B                  DATA_CARD_JSON, 139 bytes:
          {"seq":7,"libraries":[{"key":"s1","revision":1,"scope":["Books"],"cards":4,"index":{"card_version":"1.0","kind":"index","title":"Books"}}]}
```

It then queries the first page of Books, newest first:

```
Request:  type 1001 (0x03E9)
          00 01
          04 00  00 2A                  DATA_CARD_JSON, 42 bytes:
          {"library":"s1","sort":"newest","limit":2}
```

The reply carries `total` 4, the tab `All` with count 4, the facets, and two items: Second Light and The Lantern Keeper's Daughter. It also carries a `next` cursor. The next request repeats the first with `"cursor"` set to that value, and its reply holds Low Tide and The Drowned Bell, with `next` set to `null`.
