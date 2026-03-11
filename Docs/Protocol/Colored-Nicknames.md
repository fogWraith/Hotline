# Colored Nicknames Extension

The colored nicknames extension adds an optional `DATA_COLOR` field to user-related Hotline transactions, allowing per-user RGB nickname coloring. The extension is backward-compatible — clients that do not support it simply ignore the extra bytes.

## Table of Contents

- [Overview](#overview)
- [Field Type](#field-type)
- [Color Format](#color-format)
- [Opt-In Mechanism](#opt-in-mechanism)
- [Transactions That Include Color](#transactions-that-include-color)
- [User Payload Extension](#user-payload-extension)
- [Implementation Notes](#implementation-notes)

---

## Overview

The standard Hotline protocol has no provision for per-user display customisation beyond a nickname and a 16-bit icon ID. The colored nicknames extension adds a 32-bit RGB color value that clients can use to render user names in a custom color.

The extension was introduced by third-party Hotline clients and servers. It is not part of the original Hotline 1.x specification.

---

## Field Type

| ID (hex) | ID (dec) | Name | Width | Constant |
|---|---|---|---|---|
| `0x0500` | 1280 | Color | 32-bit unsigned | `HTLC_DATA_COLOR` / `HTLS_DATA_COLOR` |

Both directions (client → server and server → client) use the same field ID.

---

## Color Format

The color is a 32-bit unsigned integer sent in network byte order (big-endian), packed as `0x00RRGGBB`:

```
Bits 24-31:  Reserved (should be zero; may be used for alpha in the future)
Bits 16-23:  Red
Bits  8-15:  Green
Bits  0-7:   Blue
```

Extraction:

```c
uint8_t red   = (color & 0x00FF0000) >> 16;
uint8_t green = (color & 0x0000FF00) >> 8;
uint8_t blue  = (color & 0x000000FF);
```

A value of `-1` (0xFFFFFFFF) or absence of the field means "no color" — use the client's default rendering.

---

## Opt-In Mechanism

There is no explicit capability negotiation for colored nicknames. Clients opt in implicitly:

- **Auto detection:** When a client sends a `DATA_COLOR` field in a Set Client User Info (304) packet — i.e. the client sets its own color — the server marks that session as color-aware and includes `DATA_COLOR` in subsequent user change notifications sent to that client.
- **Server policy override:** Servers may force color on or off for all clients regardless of opt-in:
  - `"always"` — include color for all clients unconditionally
  - `"never"` — never include color for any client
  - `"auto"` (default) — include only for clients that have opted in

This per-recipient filtering means the same user change event may be sent with color to some recipients and without color to others.

---

## Transactions That Include Color

The `DATA_COLOR` field (0x0500) may appear in:

| Transaction | ID | Direction | Context |
|---|---|---|---|
| Notify of a user change | 301 | Server → Client | Broadcast when a user's state changes |
| Notify chat of a user change | 117 | Server → Client | Sent to private chat participants |
| Set client user info | 304 | Client → Server | Client sets its own nick/icon/color |
| User self-info | — | Server → Client | Server tells the logged-in user its own state |

The color field is **optional** in all of these transactions. Servers should only include it when the receiving client is known to support the extension.

---

## User Payload Extension

When color is associated with a user, the 4-byte color value is appended to the standard user info payload structure. The standard format:

| Field | Size | Description |
|---|---|---|
| User ID | 2 | Unsigned 16-bit |
| Icon ID | 2 | Unsigned 16-bit |
| User flags | 2 | Status flags (away, admin, etc.) |
| User name size | 2 | Length of name string |
| User name | N | Name string |

Extended with color:

| Field | Size | Description |
|---|---|---|
| User ID | 2 | Unsigned 16-bit |
| Icon ID | 2 | Unsigned 16-bit |
| User flags | 2 | Status flags |
| User name size | 2 | Length of name string |
| User name | N | Name string |
| **Nick color** | **4** | **Optional — `0x00RRGGBB`** |

Clients that do not support the extension will read the standard fields and ignore the trailing 4 bytes.

---

## Implementation Notes

- **Color assignment at login:** Servers may assign a default color based on account type (e.g. red for admins, no color for regular users). Per-account color overrides take precedence.
- **Color storage:** Colors are stored per-account (persistent) and can be set via the server's management interface.
- **No explicit negotiation:** Unlike capabilities-based extensions, colored nicknames rely on implicit signaling. Servers should not assume all connected clients support it.
- **Reserved bits:** The high byte (bits 24-31) is reserved. Implementations should set it to zero. Future extensions may use it for alpha transparency or other purposes.
