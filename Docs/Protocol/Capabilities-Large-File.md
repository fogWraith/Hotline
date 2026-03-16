# Large File Extension

This document describes the large-file extension to the Hotline protocol. It adds 64-bit sizing to control-plane fields (transactions) and transfer-plane headers (HTXF) while remaining backwards compatible with legacy 32-bit clients.

For the general capability negotiation mechanism, see [DATA_CAPABILITIES](capabilities.md).

## Table of Contents

- [Scope](#scope)
- [Terminology](#terminology)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
- [Encoding & Endianness](#encoding--endianness)
- [New and Extended Data Objects](#new-and-extended-data-objects)
- [Transaction Changes](#transaction-changes)
  - [Login (107)](#login-107)
  - [Get File Name List (200)](#get-file-name-list-200)
  - [Download File (202)](#download-file-202)
  - [Upload File (203)](#upload-file-203)
  - [Get File Info (206)](#get-file-info-206)
  - [Download Folder (210) / Upload Folder (213)](#download-folder-210--upload-folder-213)
- [Transfer Side-Channel (HTXF)](#transfer-side-channel-htxf)
  - [Handshake Flags and Length](#handshake-flags-and-length)
  - [Flattened File Object Fork Headers](#flattened-file-object-fork-headers)
- [Implementation Notes](#implementation-notes)
- [Notes](#notes)
- [Example: Login with Large-File Negotiation](#example-login-with-large-file-negotiation)

---

## Scope

Adds 64-bit sizing to control-plane fields (transactions) and transfer-plane headers (HTXF) while remaining backwards compatible with legacy 32-bit clients.

## Terminology

- *Client* / *Server*: Hotline peers.
- *Legacy*: Implementations that only understand 32-bit file sizes.
- *Large-file mode*: Both sides negotiated support for this extension.

## Compatibility and Negotiation

- Large-file mode is granted when the server authorises the peer via any of: the client advertises `CAPABILITY_LARGE_FILES` (bit 0 of `DATA_CAPABILITIES`) during login.
- Large-file mode could also be granted to the client if version matches an entry in a server's whitelist; or if the server's explicitly allow large-file mode for legacy clients (not recommended).
- Clients SHOULD send the 64-bit companion fields listed below; servers that have not enabled large-file mode MAY ignore those 64-bit fields and fall back to the legacy 32-bit values. Legacy servers may also produce "unknown transaction" depending on how this is implemented.
- Legacy peers (no capability and not known to support large files) are treated as 32-bit: control-plane replies clamp sizes to 32-bit, and directory listings intentionally hide files and folders whose true size exceeds the 32-bit ceiling.

## Encoding & Endianness

- All multi-byte integers are unsigned big-endian (network byte order) unless a legacy structure dictates otherwise.
- Field widths shown below refer to bit length; transmit the minimal number of bytes needed for that width (e.g. 64-bit fields are 8 bytes).

## New and Extended Data Objects

| ID (hex) | Name | Width | Notes |
|---|---|---|---|
| `0x01F1` | `DATA_FILESIZE64` | 64 | File size in bytes. Paired with legacy `DATA_FILESIZE` (32-bit) for compatibility. |
| `0x01F2` | `DATA_OFFSET64` | 64 | Offset into file (downloads/resume). Paired with legacy `DATA_OFFSET`. |
| `0x01F3` | `DATA_XFERSIZE64` | 64 | Transfer length (remaining bytes). Paired with legacy `DATA_XFERSIZE`. |
| `0x01F4` | `DATA_FOLDER_ITEM_COUNT64` | 64 | Folder item counts. Paired with legacy `DATA_FOLDER_ITEM_COUNT`. |

Send both 32-bit and 64-bit forms when large-file mode is active; legacy fields remain mandatory for compatibility.

## Transaction Changes

### Login (107)

- No additional data objects are required. Clients that advertise `CAPABILITY_LARGE_FILES` set bit 0 in `DATA_CAPABILITIES` (0x01F0); the server uses that (or a whitelist/global override) to allow large-file mode for the session. See [DATA_CAPABILITIES](capabilities.md) for the full negotiation flow.

### Get File Name List (200)

- Server: when large-file mode is active, append `DATA_FILESIZE64` immediately after each `DATA_FILE` object to provide the full byte length. The legacy `FileSize` field in `DATA_FILE` remains 32-bit and may clamp at 0xFFFFFFFF.
- Server: if large-file mode is **not** allowed for the peer, entries whose true size exceeds 0xFFFFFFFF are omitted from the listing (and folder counts skip them) to avoid advertising un-fetchable items to legacy clients.
- Client: prefer `DATA_FILESIZE64` when present; fall back to the 32-bit value otherwise.

### Download File (202)

- Request: clients MAY include `DATA_XFERSIZE64` to request a 64-bit-length subset; include `DATA_OFFSET64` for 64-bit resume offsets. Legacy `DATA_XFERSIZE`/`DATA_OFFSET` SHOULD still be sent with clamped values.
- Reply: servers include both legacy (`DATA_FILESIZE`, `DATA_OFFSET`, `DATA_XFERSIZE`) and 64-bit (`DATA_FILESIZE64`, `DATA_OFFSET64`, `DATA_XFERSIZE64`) sizes when large-file mode is active.
- Transfer side-channel MUST set the HTXF large-file flag (see below) when large-file mode is active.

### Upload File (203)

- Request: clients SHOULD send `DATA_XFERSIZE64` with the full upload size and keep `DATA_XFERSIZE` as the low 32 bits (or clamped). Servers use the 64-bit value when available.
- Reply: unchanged; refnum allocation is 32-bit.
- Transfer side-channel MUST set the HTXF large-file flag when large-file mode is active.

### Get File Info (206)

- Reply: servers include `DATA_FILESIZE64` alongside `DATA_FILESIZE` when large-file mode is active.

### Download Folder (210) / Upload Folder (213)

- Replies and progress packets include `DATA_FOLDER_ITEM_COUNT64` and `DATA_XFERSIZE64` when large-file mode is active; legacy 32-bit fields remain present.
- When large-file mode is denied for the peer, folder listings filter out oversized children; counts reflect only items visible to that legacy client.

## Transfer Side-Channel (HTXF)

### Handshake Flags and Length

The HTXF handshake is a 16-byte (or 24-byte) header sent at the start of each transfer connection:

| Offset | Size | Description |
|---|---|---|
| 0–3 | 4 | Protocol tag (`HTXF`) |
| 4–7 | 4 | Transfer reference number (from control plane) |
| 8–11 | 4 | Transfer length (legacy 32-bit) |
| 12–15 | 4 | Flags |
| 16–23 | 8 | *Optional* — 64-bit transfer length (only when `HTXF_FLAG_SIZE64` is set) |

**Flags:**

| Bit | Mask | Name | Description |
|---|---|---|---|
| 0 | `0x00000001` | `HTXF_FLAG_LARGE_FILE` | Indicates large-file mode is active for this transfer |
| 1 | `0x00000002` | `HTXF_FLAG_SIZE64` | An 8-byte length follows the 16-byte header |

- Clients set `HTXF_FLAG_LARGE_FILE` when large-file mode is negotiated or when the transfer size exceeds 32-bit. Servers mirror the flag into transfer state.
- `HTXF_FLAG_SIZE64` appends an optional 8-byte unsigned big-endian length immediately after the 16-byte header. Only send this flag/field when large-file mode is authorised. Legacy clients ignore the flag and read only the first 16 bytes.
- The server only emits/accepts `HTXF_FLAG_SIZE64` for clients already permitted for large-file mode. There is no separate toggle; allowing a client for large files also enables the SIZE64 header for that peer.
- When the total transfer length exceeds `0xFFFFFFFF`, set the legacy length field (bytes 8–11) to **zero** and carry the full length in the optional 64-bit field and in the control-plane `DATA_*64` objects. This prevents unintended clamping of the transfer.

### Flattened File Object Fork Headers

When `HTXF_FLAG_LARGE_FILE` is set, fork headers in the Flattened File Object use high/low 32-bit words to carry 64-bit lengths:

| Offset | Size | Description |
|---|---|---|
| 0–3 | 4 | Fork type (`"DATA"`, `"MACR"`, etc.) |
| 4–7 | 4 | High 32 bits of the fork length |
| 8–11 | 4 | Compression type (0 = none) |
| 12–15 | 4 | Low 32 bits of the fork length |

Info fork lengths follow the same pattern. Legacy readers that ignore the high word will see truncated sizes and should not enable large-file mode.

## Implementation Notes

- Treat all integer fields as unsigned; clamp legacy 32-bit fields instead of wrapping when conveying >4 GiB values.
- Validate HTXF flags before attempting to read the optional 64-bit length; reject SIZE64 use when the peer is not authorised for large-file mode.
- In legacy mode, omit oversized entries from directory listings and use the 32-bit ceiling for reported sizes and counts to avoid advertising un-fetchable items.
- Always send both 32-bit and 64-bit companions in large-file mode so mixed peers can continue operating; fall back to legacy values when a 64-bit counterpart is absent.

## Notes

- Always include legacy 32-bit fields for compatibility, even in large-file mode.
- Resume uses the 64-bit offset fields when present; fallback is the legacy 32-bit offset.
- Resource forks: resource fork headers also use the high/low 64-bit layout under `HTXF_FLAG_LARGE_FILE`.
- Setting HTXF length to zero for >4 GiB is required to avoid early termination by 32-bit peers.

---

## Example: Login with Large-File Negotiation

A concrete byte-level example of a client logging in with `CAPABILITY_LARGE_FILES` and the server's reply. Values are big-endian hex; the login is `admin` / `pass`, nickname `MyClient`, icon 128.

### Client → Server: Login Request (type `0x6B` = 107)

```
 Offset  Bytes                         Field
 ──────  ────────────────────────────  ─────────────────────────────────────
 HEADER (20 bytes)
 00      00 00 00 6B                   Type        = HTLC_HDR_LOGIN (107)
 04      00 00 00 01                   Seq         = 1 (task ID chosen by client)
 08      00 00 00 00                   Flags       = 0
 0C      00 00 00 31                   Size        = 49 (body length)
 10      00 00 00 31                   Check       = 49 (same as size)

 BODY (49 bytes)
 14      00 06                         FieldCount  = 6

         ── Field 1: DATA_LOGIN (0x0069) ──────────────────────────────────
 16      00 69                         ID          = DATA_LOGIN
 18      00 05                         Length      = 5
 1A      9E 9B 92 96 91                Data        = "admin" XOR 0xFF per byte

         ── Field 2: DATA_PASSWORD (0x006A) ───────────────────────────────
 1F      00 6A                         ID          = DATA_PASSWORD
 21      00 04                         Length      = 4
 23      8F 9E 8C 8C                   Data        = "pass" XOR 0xFF per byte

         ── Field 3: DATA_VERSION (0x00A0) ────────────────────────────────
 27      00 A0                         ID          = DATA_VERSION
 29      00 02                         Length      = 2
 2B      00 BE                         Data        = 190 (client version)

         ── Field 4: DATA_NICK (0x0066) ───────────────────────────────────
 2D      00 66                         ID          = DATA_NICK
 2F      00 08                         Length      = 8
 31      4D 79 43 6C 69 65 6E 74       Data        = "MyClient" (UTF-8)

         ── Field 5: DATA_ICON (0x0068) ───────────────────────────────────
 39      00 68                         ID          = DATA_ICON
 3B      00 02                         Length      = 2
 3D      00 80                         Data        = 128 (icon ID)

         ── Field 6: DATA_CAPABILITIES (0x01F0) ──────────────────────────
 3F      01 F0                         ID          = DATA_CAPABILITIES
 41      00 02                         Length      = 2
 43      00 01                         Data        = 1 (CAPABILITY_LARGE_FILES)
```

Total packet: 20 (header) + 49 (body) = **69 bytes**.

### Server → Client: Login Reply (type `0x10000` = 65536)

```
 Offset  Bytes                         Field
 ──────  ────────────────────────────  ─────────────────────────────────────
 HEADER (20 bytes)
 00      00 01 00 00                   Type        = HTLS_HDR_TASK (65536)
 04      00 00 00 01                   Seq         = 1 (echoed from client)
 08      00 00 00 00                   Flags       = 0 (0 = success, 1 = error)
 0C      00 00 00 21                   Size        = 33 (body length)
 10      00 00 00 21                   Check       = 33

 BODY (33 bytes)
 14      00 04                         FieldCount  = 4

         ── Field 1: DATA_SERVERNAME (0x00A2) ─────────────────────────────
 16      00 A2                         ID          = DATA_SERVERNAME
 18      00 09                         Length      = 9
 1A      4D 79 20 53 65 72 76 65 72    Data        = "My Server" (UTF-8)

         ── Field 2: DATA_VERSION (0x00A0) ────────────────────────────────
 23      00 A0                         ID          = DATA_VERSION
 25      00 02                         Length      = 2
 27      00 BE                         Data        = 190 (server version)

         ── Field 3: DATA_UID (0x0067) ────────────────────────────────────
 29      00 67                         ID          = DATA_UID
 2B      00 02                         Length      = 2
 2D      00 01                         Data        = 1 (assigned user ID)

         ── Field 4: DATA_CAPABILITIES (0x01F0) ──────────────────────────
 2F      01 F0                         ID          = DATA_CAPABILITIES
 31      00 02                         Length      = 2
 33      00 01                         Data        = 1 (CAPABILITY_LARGE_FILES confirmed)
```

Total packet: 20 (header) + 33 (body) = **53 bytes**.

### What This Means

The server echoed `DATA_CAPABILITIES = 0x0001` back, confirming large-file mode is active for this session. Subsequent file transactions will include 64-bit companion fields (`DATA_FILESIZE64`, `DATA_OFFSET64`, `DATA_XFERSIZE64`), and HTXF transfers will use the `HTXF_FLAG_LARGE_FILE` / `HTXF_FLAG_SIZE64` handshake flags.

If the server did **not** support large files (or denied them for this client), `DATA_CAPABILITIES` would either be absent from the reply or have bit 0 cleared — the session falls back to legacy 32-bit mode.

---

Status: draft; subject to refinement.
