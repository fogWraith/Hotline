# GIF Icons Extension

The GIF icon extension allows Hotline users to set a custom GIF avatar that other clients can download and display. The design is **pull-based**: the server never pushes unrequested GIF data. It only broadcasts a lightweight notification when an icon changes, and clients request the actual image on demand. This prevents bandwidth abuse.

## Table of Contents

- [Overview](#overview)
- [Transaction Types](#transaction-types)
- [Field Types](#field-types)
- [Transaction Details](#transaction-details)
  - [Get Icon List (1861)](#get-icon-list-1861--0x0745)
  - [Set Icon (1862)](#set-icon-1862--0x0746)
  - [Get Icon (1863)](#get-icon-1863--0x0747)
  - [Icon Change (1864)](#icon-change-1864--0x0748)
- [Data Formats](#data-formats)
  - [GIF Icon Data (0x0300)](#gif-icon-data-0x0300)
  - [Icon List Entry (0x0301)](#icon-list-entry-0x0301)
- [Interaction with Standard Icon System](#interaction-with-standard-icon-system)
- [Implementation Notes](#implementation-notes)
- [Sequence Diagram](#sequence-diagram)

---

## Overview

The standard Hotline protocol provides a 16-bit icon ID that references a sprite from a built-in icon set. The GIF icon extension supplements this by allowing users to upload and retrieve custom GIF images as avatars.

Key design principles:

- **Pull-based delivery** — the server stores GIF data per-session but never pushes it to clients. Clients must explicitly request icon data via Get Icon (1863) or Get Icon List (1861).
- **Change notification** — when a user sets or clears their icon, the server broadcasts an Icon Change (1864) packet containing only the user's ID. Clients that wish to display the updated avatar re-fetch it on demand.
- **Size limit** — servers may impose a maximum file size on uploaded GIF icons to prevent abuse.

---

## Transaction Types

| ID (hex) | ID (dec) | Name | Direction | Constant |
|---|---|---|---|---|
| `0x0745` | 1861 | Get Icon List | Client → Server | `htlcTran_IconGetList` |
| `0x0746` | 1862 | Set Icon | Client → Server | `htlcTran_IconSet` |
| `0x0747` | 1863 | Get Icon | Client → Server | `htlcTran_IconGet` |
| `0x0748` | 1864 | Icon Change | Server → Clients | `htlsTran_IconChange` |

---

## Field Types

| ID (hex) | ID (dec) | Name | Description |
|---|---|---|---|
| `0x0300` | 768 | GIF Icon Data | Raw GIF image bytes |
| `0x0301` | 769 | Icon List Entry | Packed UID + GIF data for one user |
| `0x0067` | 103 | User ID | Standard Hotline user ID field |

---

## Transaction Details

### Get Icon List (1861 / 0x0745)

- **Constant:** `htlcTran_IconGetList`
- **Initiator:** Client

Requests a full list of GIF icons for all connected users.

**Fields used in the request:** None

**Fields used in the reply:**

| ID | Field Name | Note |
|---|---|---|
| 0x0301 | Icon list entry | One per connected user |

Each icon list entry is a packed binary object (see [Icon List Entry format](#icon-list-entry-0x0301)).

---

### Set Icon (1862 / 0x0746)

- **Constant:** `htlcTran_IconSet`
- **Initiator:** Client

Sets or clears the user's GIF icon. Sending an empty data field clears the icon.

**Fields used in the request:**

| ID | Field Name | Note |
|---|---|---|
| 0x0300 | GIF Icon Data | Raw GIF data, or empty to clear |

**Fields used in the reply:** Standard task completion.

After a successful set, the server broadcasts an Icon Change (1864) notification to all connected users.

---

### Get Icon (1863 / 0x0747)

- **Constant:** `htlcTran_IconGet`
- **Initiator:** Client

Requests the GIF icon for a specific user.

**Fields used in the request:**

| ID | Field Name | Note |
|---|---|---|
| 103 | User ID | Target user whose icon is requested |

**Fields used in the reply:**

| ID | Field Name | Note |
|---|---|---|
| 103 | User ID | User the icon belongs to |
| 0x0300 | GIF Icon Data | Raw GIF image data |

---

### Icon Change (1864 / 0x0748)

- **Constant:** `htlsTran_IconChange`
- **Initiator:** Server

Broadcast to all connected users when a user sets or clears their GIF icon. This notification contains **only** the user ID — not the GIF data itself. Clients should fetch the updated icon using Get Icon (1863) if they wish to display it.

**Fields used:**

| ID | Field Name | Note |
|---|---|---|
| 103 | User ID | User whose icon changed |

---

## Data Formats

### GIF Icon Data (0x0300)

Raw GIF image bytes. Must begin with a valid GIF signature:

| Signature | Version |
|---|---|
| `GIF87a` | Original GIF specification |
| `GIF89a` | Extended GIF with animation support |

Servers should reject uploads that do not begin with one of these signatures.

### Icon List Entry (0x0301)

A packed binary object containing the user ID and GIF data for one user:

| Offset | Size | Type | Description |
|---|---|---|---|
| 0 | 2 | uint16 | User ID (big-endian) |
| 2 | 2 | uint16 | GIF data length (big-endian) |
| 4 | N | bytes | Raw GIF image data |

One Icon List Entry is sent per connected user in the Get Icon List reply. Users without a GIF icon may be omitted from the list or included with a server-provided default icon.

---

## Interaction with Standard Icon System

The GIF icon system is **independent** of the standard Hotline icon ID (`DATA_ICON` / 0x0068). The two systems coexist:

- The standard icon ID is a 16-bit integer referencing a built-in icon sprite. It is always included in user list entries and user change notifications.
- The GIF icon is raw image data transferred only through the dedicated GIF icon transactions (1861–1864).
- Clients that support GIF icons can overlay or replace the standard icon with the GIF avatar at the presentation layer.

---

## Implementation Notes

- The server stores GIF icons per-session. Icons are not persisted across reconnections (server implementation may vary).
- Servers should impose a maximum upload size (e.g. 32 KiB) and validate the GIF signature to prevent abuse.
- Get Icon (1863) requests should not reset the user's idle timer, as clients may poll for icons periodically.
- The server may provide a configurable default GIF icon for users who have not set a custom one.

---

## Sequence Diagram

```
Client A                    Server                     Client B
  |                           |                           |
  |                           |                           |
  |--- Set Icon (1862) ------>|                           |
  |    GIF data = [image]     |                           |
  |                           |--- Icon Change (1864) --->|
  |<-- Task Complete ---------|    UID = A                |
  |                           |                           |
  |                           |    [Client B wants to     |
  |                           |     display A's icon]     |
  |                           |                           |
  |                           |<-- Get Icon (1863) -------|
  |                           |    UID = A                |
  |                           |                           |
  |                           |--- Reply --------------->|
  |                           |    UID = A                |
  |                           |    GIF data = [image]     |
  |                           |                           |
```
