# Large File Extension

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document describes the large-file extension to the Hotline protocol. It adds 64-bit sizing to control-plane fields (transactions) and transfer-plane headers (HTXF) while remaining backwards compatible with legacy 32-bit clients.

For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md).

## Table of Contents

- [Scope](#scope)
- [Terminology](#terminology)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
- [Encoding & Endianness](#encoding--endianness)
- [Legacy Field Reference](#legacy-field-reference)
- [New and Extended Data Objects](#new-and-extended-data-objects)
- [Transaction Changes](#transaction-changes)
  - [Login (107)](#login-107)
  - [Get File Name List (200)](#get-file-name-list-200)
  - [Download File (202)](#download-file-202)
  - [Upload File (203)](#upload-file-203)
  - [Get File Info (206)](#get-file-info-206)
  - [Download Folder (210)](#download-folder-210)
  - [Upload Folder (213)](#upload-folder-213)
  - [Unaffected Transactions](#unaffected-transactions)
- [Transfer Side-Channel (HTXF)](#transfer-side-channel-htxf)
  - [Connection Lifecycle](#connection-lifecycle)
  - [Handshake Flags and Length](#handshake-flags-and-length)
  - [Flattened File Object (FFO)](#flattened-file-object-ffo)
  - [Flattened File Object Fork Headers](#flattened-file-object-fork-headers)
  - [Large-File Transfer Mode](#large-file-transfer-mode)
  - [File Resume (RFLT)](#file-resume-rflt)
    - [Resume Digest](#resume-digest)
  - [Folder Transfer Wire Format](#folder-transfer-wire-format)
- [Implementation Notes](#implementation-notes)
  - [Unauthorised `HTXF_FLAG_LARGE_FILE`](#unauthorised-htxf_flag_large_file)
- [Notes](#notes)
- [Example: Login with Large-File Negotiation](#example-login-with-large-file-negotiation)
- [Example: Download File with 64-bit Fields](#example-download-file-with-64-bit-fields)
- [Example: HTXF Handshake (24-byte)](#example-htxf-handshake-24-byte)

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

## Legacy Field Reference

This extension adds 64-bit companion fields alongside existing 32-bit fields. For implementors' convenience, the legacy field IDs referenced throughout this document are listed here. Full definitions are in [Hotline.md](Hotline.md).

| ID (hex) | Decimal | Name | Width | Used In |
|---|---|---|---|---|
| `0x006B` | 107 | `DATA_REFNUM` | 32-bit | Transfer reference number (server-assigned) |
| `0x006C` | 108 | `DATA_TRANSFERSIZE` | 32-bit | Transfer size (remaining bytes) |
| `0x00C8` | 200 | `DATA_FILE_NAME_WITH_INFO` | Variable | File entry in directory listings |
| `0x00CB` | 203 | `DATA_FILE_RESUME_DATA` | Variable | RFLT resume structure |
| `0x00CC` | 204 | `DATA_FILE_TRANSFER_OPTIONS` | Variable | Transfer option flags (value `2` = resume) |
| `0x00CF` | 207 | `DATA_FILESIZE` | 32-bit | File size |
| `0x00DC` | 220 | `DATA_FOLDER_ITEM_COUNT` | 16-bit | Folder item count |

### DATA_FILE_NAME_WITH_INFO Structure

`DATA_FILE_NAME_WITH_INFO` (`0x00C8`) is a packed binary field returned in Get File Name List (200) responses. One field is emitted per file or folder entry:

| Offset | Size | Field | Description |
|---|---|---|---|
| 0 | 4 | Type | Four-character type code (e.g. `"TEXT"`, `"SITD"`). Folders use `"fldr"`. |
| 4 | 4 | Creator | Four-character creator code. Folders use `"\x00\x00\x00\x00"`. |
| 8 | 4 | FileSize | 32-bit file size in bytes. For folders, this is the child item count. Clamped to `0xFFFFFFFF` for large files. |
| 12 | 4 | Reserved | Always zero. |
| 16 | 2 | NameScript | Script/encoding identifier (typically `0`). |
| 18 | 2 | NameSize | Length of the Name field in bytes (N). |
| 20 | N | Name | File or folder name (variable length). |

Total size: 20 + N bytes.

## New and Extended Data Objects

| ID (hex) | Name | Width | Notes |
|---|---|---|---|
| `0x01F1` | `DATA_FILESIZE64` | 64 | File size in bytes. Paired with legacy `DATA_FILESIZE` (32-bit) for compatibility. |
| `0x01F2` | `DATA_OFFSET64` | 64 | Offset into file (downloads/resume). Paired with legacy `DATA_OFFSET`. |
| `0x01F3` | `DATA_XFERSIZE64` | 64 | Transfer length (remaining bytes). Paired with legacy `DATA_XFERSIZE`. |
| `0x01F4` | `DATA_FOLDER_ITEM_COUNT64` | 64 | Folder item counts. Paired with legacy `DATA_FOLDER_ITEM_COUNT` (16-bit). While a 64-bit width is far larger than any realistic item count, this matches the convention used by the other companion fields for consistency. |
| `0x01FA` | `DATA_PARTIAL_DIGEST` | 320 | Resume digest of a partial upload: an 8-byte window length followed by a 32-byte SHA-256. Sent in the Upload File (203) reply, and echoed by the client in the `HTXF_FLAG_RESUME` handshake extension. See [Resume Digest](#resume-digest). Not `0x01F5`, which would have continued this block — the large-file allocation ends at `0x01F4` and the voice extension holds `0x01F5`–`0x01F9`. |

Send both legacy and 64-bit forms when large-file mode is active; legacy fields remain mandatory for compatibility.

## Transaction Changes

### Login (107)

- No additional data objects are required. Clients that advertise `CAPABILITY_LARGE_FILES` set bit 0 in `DATA_CAPABILITIES` (0x01F0); the server uses that (or a whitelist/global override) to allow large-file mode for the session. See [DATA_CAPABILITIES](Capabilities.md) for the full negotiation flow.

### Get File Name List (200)

- Server: when large-file mode is active, append `DATA_FILESIZE64` immediately after each `DATA_FILE` object to provide the full byte length. The legacy `FileSize` field in `DATA_FILE` remains 32-bit and may clamp at 0xFFFFFFFF.
- Server: if large-file mode is **not** allowed for the peer, entries whose true size exceeds 0xFFFFFFFF are omitted from the listing (and folder counts skip them) to avoid advertising un-fetchable items to legacy clients.
- Client: prefer `DATA_FILESIZE64` when present; fall back to the 32-bit value otherwise.

### Download File (202)

- Request: clients MAY include `DATA_XFERSIZE64` to request a 64-bit-length subset; include `DATA_OFFSET64` for 64-bit resume offsets. Legacy `DATA_XFERSIZE`/`DATA_OFFSET` SHOULD still be sent with clamped values.
- Reply: servers include both legacy (`DATA_FILESIZE`, `DATA_OFFSET`, `DATA_XFERSIZE`) and 64-bit (`DATA_FILESIZE64`, `DATA_OFFSET64`, `DATA_XFERSIZE64`) sizes when large-file mode is active.
- Transfer side-channel MUST set the HTXF large-file flag (see below) when large-file mode is active.

### Upload File (203)

- Request: clients SHOULD send `DATA_XFERSIZE64` with the full upload size. The legacy `DATA_XFERSIZE` MUST be clamped to `0xFFFFFFFF` when the true size exceeds 32 bits. Servers use the 64-bit value when available.
- Reply: servers include the refnum (32-bit) and, when resuming a partially transferred file, include both the legacy `DATA_OFFSET` (clamped) and `DATA_OFFSET64` fields, plus `DATA_XFERSIZE` and `DATA_XFERSIZE64` echoing the expected transfer size.
- Reply: in large-file mode, servers SHOULD also include `DATA_PARTIAL_DIGEST` so the client can confirm the partial belongs to the file it is about to resume. A server that cannot produce one omits the field, and the client then sends the whole file. See [Resume Flow (Upload)](#resume-flow-upload).
- Transfer side-channel MUST set the HTXF large-file flag when large-file mode is active.

### Get File Info (206)

- Reply: servers include `DATA_FILESIZE64` alongside `DATA_FILESIZE` when large-file mode is active.

### Download Folder (210)

- Reply: servers include `DATA_FOLDER_ITEM_COUNT64` and `DATA_XFERSIZE64` alongside their legacy counterparts when large-file mode is active.
- When large-file mode is denied for the peer, folder listings filter out oversized children; counts reflect only items visible to that legacy client.

### Upload Folder (213)

- Request: clients SHOULD send `DATA_FOLDER_ITEM_COUNT64` and `DATA_XFERSIZE64` alongside their legacy counterparts. Servers prefer the 64-bit values when present.
- Reply: servers include the refnum (32-bit). The server stores the folder item count and transfer size for progress tracking.

### Unaffected Transactions

The following transactions do not require large-file changes because they operate on file metadata or path information rather than file content or size:

- **Delete File (204)** — deletes by path; no size fields involved.
- **Move File (205)** — moves by path; no size fields involved.
- **Rename File (not defined in this extension)** — renames by path; no size fields involved.
- **Set File Info (207)** — writes metadata (comment, type, creator); does not reference file size.

Implementations MUST NOT reject these transactions based on whether large-file mode is active.

## Transfer Side-Channel (HTXF)

### Connection Lifecycle

File transfers use a dedicated TCP connection, separate from the control connection used for transactions. The flow is:

1. **Control plane setup:** The client sends a download or upload transaction on the control connection. The server replies with a `DATA_REFNUM` (4-byte reference number) and transfer size fields.
2. **HTXF connection:** The client opens a new TCP connection to **port + 1** (e.g. if the server's control port is 5500, the transfer port is 5501). For TLS sessions, a separate TLS transfer port is used (typically `tls_port + 1`, or as configured).
3. **Handshake:** The client writes the HTXF handshake (16 or 24 bytes; see below). The server reads the handshake, matches the reference number to a pending transfer, and clears the handshake deadline.
4. **Data exchange:** Depending on the transfer type, data flows in one direction:
   - *Download:* server writes to client.
   - *Upload:* client writes to server.
5. **Completion:** The data exchange finishes when the expected byte count is reached. The connection is then closed. There is no explicit "transfer complete" message.

**Timeout:** Servers SHOULD impose a handshake timeout (e.g. 10 seconds) on accepted transfer connections. If the client does not send a valid handshake within this window, the server closes the connection. The deadline is cleared once the handshake is successfully read, allowing the data phase to run without a timeout.

**Reference number lifecycle:** The reference number is single-use. Once the transfer completes (or fails), the server deletes the reference. A client that reconnects with the same reference number after completion will be rejected.

### Handshake Flags and Length

The HTXF handshake is a 16-byte base header sent at the start of each transfer connection, optionally followed by fixed-size extension blocks:

| Offset | Size | Description |
|---|---|---|
| 0–3 | 4 | Protocol tag (`HTXF`) |
| 4–7 | 4 | Transfer reference number (from control plane) |
| 8–11 | 4 | Transfer length (legacy 32-bit) |
| 12–15 | 4 | Flags |
| 16 | 8 | *Optional* — 64-bit transfer length (only when `HTXF_FLAG_SIZE64` is set) |
| next | 40 | *Optional* — resume digest (only when `HTXF_FLAG_RESUME` is set); see [Resume Digest](#resume-digest) |

Extension blocks appear in flag-bit order, so the handshake is 16, 24, 56 or 64 bytes. Each is present if and only if its flag is set: there is no other way to detect one, and a server cannot recover from a client that sets a flag and omits the block, or omits the flag and sends one.

Blocks are fixed-size for that reason. A future digest algorithm needs a new flag bit rather than a variable-length block, which an implementation that did not recognise it could not skip.

Extension blocks are addressed to the peer that reads the handshake, and are **not** part of the payload. Where a server splices two peers together rather than storing the file — the user-to-user relay of the [messaging extension](Capabilities-Messaging.md#handshake-flags-on-the-relay) — it MUST consume every block the flags declare and forward only what follows. A block left on the stream is delivered to the far peer as file content.

Blocks are also sent **in the clear**, ahead of any transport encryption negotiated for the payload. An implementation that reads them after wrapping the connection decodes framed payload as though it were the block.

**Flags:**

| Bit | Mask | Name | Description |
|---|---|---|---|
| 0 | `0x00000001` | `HTXF_FLAG_LARGE_FILE` | Indicates large-file mode is active for this transfer |
| 1 | `0x00000002` | `HTXF_FLAG_SIZE64` | An 8-byte length follows the 16-byte header |
| 2 | `0x00000004` | `HTXF_FLAG_RESUME` | This large-file upload continues the partial the server described, rather than replacing it |

- Clients set `HTXF_FLAG_LARGE_FILE` when large-file mode is negotiated. Servers mirror the flag into transfer state.
- `HTXF_FLAG_SIZE64` appends an optional 8-byte unsigned big-endian length immediately after the 16-byte header. Only send this flag/field when large-file mode is authorised. Legacy clients ignore the flag and read only the first 16 bytes.
- **`HTXF_FLAG_RESUME`** applies only to a large-file upload, and only after the server has quoted a resume offset in its Upload File (203) reply. It MUST be accompanied by `HTXF_FLAG_LARGE_FILE`, and appends a 40-byte [resume digest](#resume-digest) to the handshake. A server MUST reject it on any other transfer type, and MUST reject it on a transfer for which it issued no offset. See [Resume Flow (Upload)](#resume-flow-upload).
- **Authorisation:** A client MUST NOT set any of these flags unless the server confirmed `CAPABILITY_LARGE_FILES` in the login reply. Advertising the capability is not the same as being granted it, and a client MUST NOT infer the grant from having asked for it. A server MUST reject a transfer whose handshake sets any of them when the peer is not authorised, and MUST NOT attempt to interpret the payload that follows. See [Error handling](#implementation-notes).
- **Why every flag is gated:** the flags do not request a capability, they *declare how the bytes that follow are framed*. An unauthorised `HTXF_FLAG_SIZE64` desynchronises the stream immediately, because the server either consumes eight payload bytes as a length or reads a length that was never sent. `HTXF_FLAG_LARGE_FILE` and `HTXF_FLAG_RESUME` are more dangerous precisely because they do not: the server accepts a well-formed transfer, stores the wrong bytes, and reports success. The first decides whether the payload is wrapped; the second decides whether it starts at byte zero. Getting either wrong yields a file of plausible size that is corrupt inside. See [Unauthorised `HTXF_FLAG_LARGE_FILE`](#unauthorised-htxf_flag_large_file).
- **Flag relationship:** `HTXF_FLAG_LARGE_FILE` is the base flag; both `HTXF_FLAG_SIZE64` and `HTXF_FLAG_RESUME` are only valid when it is also set, and authorising a client for large files authorises all three. A client MAY set `HTXF_FLAG_LARGE_FILE` alone — without `HTXF_FLAG_SIZE64` for transfers where a 32-bit length suffices, and without `HTXF_FLAG_RESUME` whenever it is sending a whole file. The server determines the transfer direction from its stored transfer state, not from which flags are present.
- When the total transfer length exceeds `0xFFFFFFFF`, set the legacy length field (bytes 8–11) to **zero** and carry the full length in the optional 64-bit field and in the control-plane `DATA_*64` objects. This prevents unintended clamping of the transfer.

### Flattened File Object (FFO)

File downloads (and folder item downloads) transmit files as a **Flattened File Object** (FFO), which encapsulates file metadata and data in a single self-describing stream. The FFO consists of a fixed header followed by one or more fork sections (INFO, DATA, and optionally MACR for Mac resource forks).

#### FFO Header (24 bytes)

| Offset | Size | Field | Description |
|---|---|---|---|
| 0 | 4 | Format | Magic: `"FILP"` (`0x46494C50`) |
| 4 | 2 | Version | Always `0x0001` |
| 6 | 16 | Reserved | Must be zero |
| 22 | 2 | ForkCount | Number of fork sections that follow: `2` (INFO + DATA) or `3` (INFO + DATA + MACR) |

#### FFO Wire Layout

A complete FFO for a two-fork file (no resource fork) is laid out as:

```
[FFO Header (24 bytes)]
  ├── [INFO Fork Header (16 bytes)]
  ├── [INFO Fork Data   (variable; see below)]
  ├── [DATA Fork Header (16 bytes)]
  └── [DATA Fork Data   (variable; raw file content)]
```

If a resource fork is present (`ForkCount = 3`), a MACR fork header and data follow after the DATA fork.

Each fork section consists of a **Fork Header** (16 bytes; see next section) immediately followed by the fork's data bytes. The fork header's `DataSize` (or 64-bit equivalent in large-file mode) specifies how many bytes of data follow.

#### INFO Fork Data

The INFO fork carries file metadata. Its structure is:

| Offset | Size | Field | Description |
|---|---|---|---|
| 0 | 4 | Platform | Platform identifier: `"AMAC"` (Mac) or `"MWIN"` (Windows) |
| 4 | 4 | TypeSignature | Four-character file type (e.g. `"TEXT"`, `"APPL"`). Folders: `"fldr"`. |
| 8 | 4 | CreatorSignature | Four-character creator code (e.g. `"MSWD"`). |
| 12 | 4 | Flags | File flags (bitfield). |
| 16 | 4 | PlatformFlags | Platform-specific flags. |
| 20 | 32 | Reserved | Must be zero. |
| 52 | 8 | CreateDate | Creation timestamp. |
| 60 | 8 | ModifyDate | Modification timestamp. |
| 68 | 2 | NameScript | Script/encoding identifier (typically `0`). |
| 70 | 2 | NameSize | Length of filename in bytes (N; max 128). |
| 72 | N | Name | Filename (N bytes). |
| 72+N | 2 | CommentSize | Length of comment in bytes (C). May be `0`. |
| 74+N | C | Comment | File comment (C bytes; optional). |

**INFO fork data total size:** 74 + N + C bytes.

The INFO fork header's `DataSize` MUST equal this total.

### Flattened File Object Fork Headers

The legacy fork header is a 16-byte structure used to introduce each fork (INFO, DATA, MACR) in a Flattened File Object:

| Offset | Size | Legacy field name | Legacy purpose |
|---|---|---|---|
| 0–3 | 4 | Fork type | Four-character code: `"INFO"`, `"DATA"`, or `"MACR"` |
| 4–7 | 4 | Compression type | Compression algorithm (0 = none) |
| 8–11 | 4 | Reserved | Reserved, must be zero |
| 12–15 | 4 | Data size | Fork length in bytes (32-bit) |

When `HTXF_FLAG_LARGE_FILE` is set, the same 16-byte structure is reinterpreted. The Compression type field (offset 4–7) is repurposed to carry the **high** 32 bits of the fork length, while the Data size field (offset 12–15) carries the **low** 32 bits. The Reserved field (offset 8–11) remains zero.

| Offset | Size | Large-file interpretation |
|---|---|---|
| 0–3 | 4 | Fork type (unchanged) |
| 4–7 | 4 | **High** 32 bits of the 64-bit fork length |
| 8–11 | 4 | Reserved (zero) |
| 12–15 | 4 | **Low** 32 bits of the 64-bit fork length |

To reconstruct the full fork size: `fork_length = (high << 32) | low`.

The binary layout is **identical** in both modes; only the interpretation of offset 4–7 changes. Compression is not used in practice (always zero), which makes this repurposing safe. Legacy readers that do not check high bits will see only the low 32-bit size — a truncated value — and should not enable large-file mode.

Info fork and resource fork (`"MACR"`) headers follow the same pattern.

### Large-File Transfer Mode

The large-file extension changes the transfer-plane behaviour depending on direction:

#### Downloads (Server → Client)

Downloads use the standard FFO wire format (header + fork headers + fork data). In large-file mode, fork headers carry 64-bit sizes via the split encoding described above (high 32 bits in `CompressionType`, low 32 bits in `DataSize`). The overall stream structure is unchanged — clients simply decode fork sizes with the 64-bit formula.

#### Uploads (Client → Server)

In large-file mode (`HTXF_FLAG_LARGE_FILE` set), uploads send **raw file data only** — no FFO wrapper. The client writes the file content directly after the HTXF handshake. The server determines the upload size from the handshake: it prefers `DataSize64` (from the 24-byte handshake when `HTXF_FLAG_SIZE64` is set) and falls back to the 32-bit `DataSize` (bytes 8–11) when the 64-bit field is absent.

No file metadata (INFO fork) is transmitted for large-file uploads. The server constructs metadata from the filesystem (type, creator, timestamps) or uses defaults.

Large-file uploads MAY be resumed, using `HTXF_FLAG_RESUME` and a [resume digest](#resume-digest) rather than RFLT — see [Resume Flow (Upload)](#resume-flow-upload). A server MUST replace any existing partial when the flag is absent: without it the client is sending the whole file, and appending to what a previous attempt left behind produces a file of plausible size with the wrong bytes in the middle.

#### Legacy Uploads

Without `HTXF_FLAG_LARGE_FILE`, uploads use the traditional FFO-wrapped format: the client sends a complete Flattened File Object (header + INFO fork + DATA fork + optional MACR fork). Fork sizes are 32-bit.

### File Resume (RFLT)

File resume uses the RFLT (Resume FiLe Transfer) structure, sent in `DATA_FILE_RESUME_DATA` (`0x00CB`). It tells the server which fork offsets the client already has.

RFLT covers every resume except one: a resumed large-file *upload*, which has no forks for RFLT to describe and uses `HTXF_FLAG_RESUME` with a single offset and a digest instead. See [Resume Flow (Upload)](#resume-flow-upload).

#### RFLT Header (42 bytes)

| Offset | Size | Field | Description |
|---|---|---|---|
| 0 | 4 | Format | Magic: `"RFLT"` (`0x52464C54`) |
| 4 | 2 | Version | Always `0x0001` |
| 6 | 34 | Reserved | Must be zero |
| 40 | 2 | ForkCount | Number of fork resume entries that follow (typically `2` or `3`) |

#### Fork Resume Entry (16 bytes each)

| Offset | Size | Field | Description |
|---|---|---|---|
| 0 | 4 | ForkType | Four-character fork type: `"DATA"`, `"INFO"`, or `"MACR"` |
| 4 | 4 | DataSize | Byte offset to resume from (32-bit). Represents how many bytes the client already has for this fork. |
| 8 | 4 | ReservedA | Must be zero |
| 12 | 4 | ReservedB | Must be zero |

**Total RFLT size:** 42 + (16 × ForkCount) bytes.

#### Resume Flow (Download)

1. Client sends Download File (202) with `DATA_FILE_RESUME_DATA` containing an RFLT listing partial fork sizes, and `DATA_FILE_TRANSFER_OPTIONS` set to `2` (resume mode).
2. Client MAY include `DATA_OFFSET64` for 64-bit resume offsets.
3. Server calculates the remaining transfer size and replies with updated `DATA_XFERSIZE` / `DATA_XFERSIZE64`.
4. On the HTXF connection, the server sends the FFO starting from the resume point. For the DATA fork, this means skipping the already-transferred bytes and sending only the remainder.

#### Resume Flow (Upload)

Resume for legacy uploads follows a similar pattern: the server replies to Upload File (203) with `DATA_FILE_RESUME_DATA` containing the RFLT for the partial file, and the client adjusts its FFO output accordingly.

Large-file uploads resume through `HTXF_FLAG_RESUME` instead. RFLT does not fit them: it carries an offset per fork, and a large-file upload has no forks — it is one contiguous stream, so a single offset describes it completely. RFLT also carries no way for the two ends to check that they agree about the bytes already transferred, which a raw stream needs more than a flattened one: there is no INFO fork naming the file, so nothing but the path distinguishes one upload from another.

The flow is:

1. **Client** sends Upload File (203) with `DATA_FILE_TRANSFER_OPTIONS` = `2`. Note that `DATA_XFERSIZE` is not sent on a resume request.
2. **Server** looks for a partial. If one exists it replies with `DATA_FILE_RESUME_DATA` (RFLT, for legacy compatibility) and, in large-file mode, `DATA_OFFSET64` carrying the partial's exact length and `DATA_PARTIAL_DIGEST` carrying the [resume digest](#resume-digest) of what it holds. It MUST remember the offset it quoted, at full width — the RFLT copy clamps above 4 GiB and cannot be used for step 5.
3. **Client** computes the same digest over its own copy of the file, up to the quoted offset, and compares.
   - **Equal:** the two ends hold the same bytes; the client MAY resume.
   - **Different, or `DATA_PARTIAL_DIGEST` absent:** the client MUST NOT resume, and sends the whole file instead. This is an ordinary outcome, not an error — the local file has changed since the failed attempt — and nothing needs to be reported to the user beyond the upload taking longer.
4. **Client** opens the HTXF connection:
   - **To resume:** set `HTXF_FLAG_RESUME` alongside `HTXF_FLAG_LARGE_FILE`, append the 40-byte digest it just verified as the handshake's resume block, set the handshake length to the **remaining** bytes (total minus the quoted offset), and send the file from that offset onward.
   - **To send the whole file:** omit `HTXF_FLAG_RESUME` and proceed as a normal large-file upload. The server discards the partial.
5. **Server** verifies before writing a byte. It MUST confirm the partial still exists, is still exactly as long as the offset it quoted in step 2, and that the digest the client echoed matches one the server recomputes **now** from the partial. If any check fails, the transfer MUST be refused and the partial MUST be left untouched, so that the client can retry without the flag.
6. **Server** appends the payload to the partial and completes the upload.

**Why the flag is required.** Steps 3 and 4 are the reason this cannot be settled on the control plane. The server learns in step 1 that the client is *interested* in resuming, but the client's actual decision is made after it sees the offset and the digest, and reaches the server only when the transfer connection opens. A server that infers resume from the step 1 request will append a complete file to a stale fragment whenever the client changes its mind.

**Why the digest is checked twice.** The client's check in step 3 is what makes the mechanism usable: it lets an honest client discover the mismatch before uploading anything, and fall back silently. The server's check in step 5 is what makes it trustworthy: a client that skips step 3, through a bug or an oversight, is refused rather than allowed to corrupt a file. Neither check alone is sufficient, and neither defends against a client that deliberately lies — such a client can send wrong bytes for the remainder just as easily, and does not need a resume to do it.

**Why the server recomputes rather than reusing.** A partial's length matching the quoted offset does not mean its content is untouched; something may have rewritten it in place. Recomputing at step 5 costs one window and removes the assumption.

#### Resume Digest

`DATA_PARTIAL_DIGEST` and the `HTXF_FLAG_RESUME` handshake block carry the same 40-byte structure:

| Offset | Size | Field |
|---|---|---|
| 0 | 8 | Window length, big-endian |
| 8 | 32 | SHA-256 |

The digest is computed over the **last `min(offset, 65536)` bytes** of the partial, with the offset prepended to the hash input as an 8-byte big-endian integer:

```
window  = min(offset, 65536)
digest  = SHA-256( be64(offset) || partial[offset-window : offset] )
```

The offset is hashed in so a digest is only ever valid at the length it was produced at; two partials sharing a trailing window at different lengths do not collide, and a digest cannot be replayed from an earlier, shorter attempt.

Servers MAY use a window larger than 65536 and MUST state it in the window-length field; clients MUST hash the window length they are given rather than assuming the default.

**This is a strong check, not a proof.** It samples a window rather than the whole prefix, for two reasons. The digest has to be produced inside the Upload File reply, on the control connection that also carries chat and the user list, so hashing gigabytes there would stall the session. Computing it during the original upload and persisting it instead would put a second artifact beside every partial, to be cleaned up wherever partials are removed, and would need a crash-consistency story for a stored length that outlives its bytes.

Two files that differ at all will differ within 64 KiB of an arbitrary offset with overwhelming probability. What the window misses is a file identical in that window and different earlier — which describes an append-only file, where resuming is the correct thing to do anyway.

### Folder Transfer Wire Format

Folder transfers stream multiple files over a single HTXF connection using a per-file framing protocol. The interaction is a dialog: one side sends a file header, the other replies with an action code, then data flows.

#### Folder Download (Server → Client)

For each file or directory in the folder (walked recursively):

1. **Server sends FileHeader (variable length):**

   | Offset | Size | Field | Description |
   |---|---|---|---|
   | 0 | 2 | HeaderSize | Total size of Type + FilePath fields in bytes |
   | 2 | 2 | Type | `0x0000` = file, `0x0001` = directory |
   | 4 | variable | FilePath | Encoded path (relative to folder root) |

2. **Client replies with a 2-byte action code:**

   | Value | Name | Meaning |
   |---|---|---|
   | `0x0001` | `DlFldrActionSendFile` | Send this file's data |
   | `0x0002` | `DlFldrActionResumeFile` | Resume a partial transfer (client sends RFLT) |
   | `0x0003` | `DlFldrActionNextFile` | Skip this file |

3. **If `SendFile` or `ResumeFile`:** the server writes:
   - **Transfer size** (4 bytes, big-endian): total bytes for this file's FFO.
   - **Flattened File Object**: complete FFO (header + forks) for the file. In large-file mode, fork headers use the 64-bit encoding.

4. **If `ResumeFile`:** the client sends its RFLT data before the server responds with the adjusted FFO.

5. **If the entry is a directory:** only the FileHeader is sent (no FFO). The client creates the directory locally.

#### Folder Upload (Client → Server)

For each file or directory the client wishes to upload:

1. **Client sends a file entry header:**

   | Offset | Size | Field | Description |
   |---|---|---|---|
   | 0 | 2 | DataSize | Total size of the remaining fields in this entry |
   | 2 | 2 | IsFolder | `0x0000` = file, `0x0001` = directory |
   | 4 | 2 | PathItemCount | Number of path segments |
   | 6 | variable | PathSegments | Encoded path segments (see below) |

   Each path segment is encoded as:

   | Size | Field |
   |---|---|
   | 2 | Segment length (N) |
   | N | Segment name bytes |

2. **Server replies with a 2-byte action code:**

   | Value | Meaning |
   |---|---|
   | `0x0001` | Send file data now |
   | `0x0002` | Resume — server sends RFLT, then client sends remaining data |
   | `0x0003` | Skip (file already exists and is complete) |

3. **If action is `0x0002` (resume):** the server sends:
   - **Resume data size** (2 bytes): length of the RFLT structure.
   - **RFLT data**: complete RFLT structure for the partial file.

4. **Client sends file data** (for action `0x0001` or after processing `0x0002`):
   - **Transfer size** (4 bytes, big-endian).
   - **Flattened File Object**: complete FFO for the file.

5. **If the entry is a directory:** the server creates the directory; no file data is exchanged.

## Implementation Notes

- Treat all integer fields as unsigned; clamp legacy 32-bit fields instead of wrapping when conveying >4 GiB values.
- **Clamping vs. zeroing:** control-plane legacy fields (`DATA_FILESIZE`, `DATA_XFERSIZE`, etc.) should be **clamped** to `0xFFFFFFFF`. The HTXF handshake legacy length field (bytes 8–11) should be set to **zero** when the transfer size exceeds 32 bits. The distinction matters: clamped control-plane values let legacy clients display approximate sizes, while a zeroed HTXF length prevents a legacy peer from attempting a partial read and treating it as complete.
- Validate HTXF flags before attempting to read the optional 64-bit length. Servers MUST reject any of `HTXF_FLAG_LARGE_FILE`, `HTXF_FLAG_SIZE64` or `HTXF_FLAG_RESUME` when the peer is not authorised for large-file mode.
- In legacy mode, omit oversized entries from directory listings and use the 32-bit ceiling for reported sizes and counts to avoid advertising un-fetchable items.
- Always send both legacy and 64-bit companion fields in large-file mode so mixed peers can continue operating; fall back to legacy values when a 64-bit counterpart is absent.
- `DATA_FILESIZE64` in Get File Name List (200) is a **separate transaction field** appended to the response field list after each `DATA_FILE` field — it is not embedded inside the `DATA_FILE` binary blob. Clients parse it by reading the field list sequentially: each `DATA_FILESIZE64` corresponds to the most recently parsed `DATA_FILE`.
- Resume: the 64-bit offset (`DATA_OFFSET64`) determines the byte position to resume from. The 64-bit transfer length in the HTXF handshake reflects the **remaining** bytes to transfer (total size minus offset), not the full file size. This holds for a resumed large-file upload too: a server sizing the copy from the total waits for bytes the client already sent and will never send again.
- **Error handling:** Servers MUST return an error reply (Flags = 1 in the transaction header) if a client requests a large-file operation but is not authorised for large-file mode. For HTXF, the server MUST close the transfer connection if any of `HTXF_FLAG_LARGE_FILE`, `HTXF_FLAG_SIZE64` or `HTXF_FLAG_RESUME` is set by an unauthorised peer. A server MUST NOT silently ignore an unauthorised flag and fall back to legacy framing: the flag describes the payload, so ignoring it means parsing the stream one way while the peer wrote it another. Implementations SHOULD log the rejection at warning level, naming the offending flag and the file, so that the client author has something actionable.

- **`DATA_CAPABILITIES` width:** The `DATA_CAPABILITIES` field is typically 2 bytes (16 bits), expandable to 8. Only bit 0 (`CAPABILITY_LARGE_FILES`) is defined by *this* extension; other bits are allocated by other extensions and a peer will routinely advertise several at once (see [DATA_CAPABILITIES](Capabilities.md) for the registry). Implementations MUST mask for bit 0 rather than comparing the whole field, and MUST ignore bits they do not recognise rather than rejecting the field.

### Unauthorised `HTXF_FLAG_LARGE_FILE`

This case has its own section because the consequence of getting it wrong is not a failed transfer — it is a corrupt file that every party reports as healthy.

The two transfer directions are framed differently (see [Large-File Transfer Mode](#large-file-transfer-mode)): a large-file *download* keeps the FFO wrapper and merely widens the fork headers, while a large-file *upload* has no wrapper at all. An implementation that builds the download side first and assumes the upload side mirrors it will set `HTXF_FLAG_LARGE_FILE` and still send a complete FFO.

A server that honours the flag from an unauthorised peer then writes that FFO to disk as file content. The `FILP` header, the INFO fork and the fork headers become the leading bytes of the stored file, and the tail is truncated by however many bytes the wrapper occupied — the server stops at the announced length. For a file named `nyx-windows-amd64.exe` the wrapper is 151 bytes: 24 for the FFO header, 16 for the INFO fork header, 95 for the INFO fork itself (74 fixed + 21 for the name, with no comment), and 16 for the DATA fork header.

The resulting file is close enough to the expected size to survive a glance, and is corrupt from byte zero. It uploads without error, lists correctly, and downloads without error — the server wraps the corrupt bytes in a valid FFO and delivers them faithfully, so even a stock Hotline 1.2.3 accepts the transfer. Nothing in either peer's logs distinguishes it from a healthy round trip. The first symptom is a user reporting that a downloaded file will not open.

To identify a file damaged this way, read its first four bytes. `46 49 4C 50` (`FILP`) in place of the file's own signature is conclusive. The original content begins at the end of the wrapper and is short by that many bytes at the tail, so the file is recoverable only if the sender still has it.

Rejecting the unauthorised flag converts this into a refused transfer at the handshake, before a single payload byte is read.

## Notes

- Always include legacy fields for compatibility, even in large-file mode.
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

## Example: Download File with 64-bit Fields

A client requests a 5 GiB file in large-file mode. The server replies with both legacy and 64-bit fields. Values are big-endian hex.

### Client → Server: Download File (type `0x00CA` = 202)

```
 Offset  Bytes                         Field
 ──────  ────────────────────────────  ─────────────────────────────────────
 HEADER (20 bytes)
 00      00 00 00 CA                   Type        = HTLC_HDR_GET_FILE (202)
 04      00 00 00 02                   Seq         = 2
 08      00 00 00 00                   Flags       = 0
 0C      00 00 00 19                   Size        = 25 (body length)
 10      00 00 00 19                   Check       = 25

 BODY (25 bytes)
 14      00 02                         FieldCount  = 2

         ── Field 1: DATA_FILENAME (0x00C9) ───────────────────────────────
 16      00 C9                         ID          = DATA_FILENAME
 18      00 0D                         Length      = 13
 1A      6C 61 72 67 65 66 69 6C
         65 2E 69 73 6F                Data        = "largefile.iso"

         ── Field 2: DATA_FILEPATH (0x00C6) ───────────────────────────────
 27      00 C6                         ID          = DATA_FILEPATH
 29      00 02                         Length      = 2
 2B      00 00                         Data        = root (empty path)
```

### Server → Client: Download File Reply

```
 Offset  Bytes                         Field
 ──────  ────────────────────────────  ─────────────────────────────────────
 HEADER (20 bytes)
 00      00 01 00 00                   Type        = HTLS_HDR_TASK (65536)
 04      00 00 00 02                   Seq         = 2
 08      00 00 00 00                   Flags       = 0
 0C      00 00 00 40                   Size        = 64 (body length)
 10      00 00 00 40                   Check       = 64

 BODY (64 bytes)
 14      00 07                         FieldCount  = 7

         ── Field 1: DATA_XFERSIZE (0x006C) ──────────────────────────────
 16      00 6C                         ID          = DATA_XFERSIZE
 18      00 04                         Length      = 4
 1A      FF FF FF FF                   Data        = 0xFFFFFFFF (clamped — real size > 4 GiB)

         ── Field 2: DATA_XFERSIZE64 (0x01F3) ────────────────────────────
 1E      01 F3                         ID          = DATA_XFERSIZE64
 20      00 08                         Length      = 8
 22      00 00 00 01 40 00 00 66       Data        = 5,368,709,222 (5 GiB FFO: header + forks + data)

         ── Field 3: DATA_FILESIZE (0x00CF) ──────────────────────────────
 2A      00 CF                         ID          = DATA_FILESIZE
 2C      00 04                         Length      = 4
 2E      FF FF FF FF                   Data        = 0xFFFFFFFF (clamped)

         ── Field 4: DATA_FILESIZE64 (0x01F1) ────────────────────────────
 32      01 F1                         ID          = DATA_FILESIZE64
 34      00 08                         Length      = 8
 36      00 00 00 01 40 00 00 00       Data        = 5,368,709,120 (5 GiB raw data)

         ── Field 5: DATA_REFNUM (0x006B) ────────────────────────────────
 3E      00 6B                         ID          = DATA_REFNUM
 40      00 04                         Length      = 4
 42      00 00 AB CD                   Data        = 0x0000ABCD (transfer reference)

         ── Field 6: DATA_WAITING_COUNT (0x006F) ─────────────────────────
 46      00 6F                         ID          = DATA_WAITING_COUNT
 48      00 02                         Length      = 2
 4A      00 00                         Data        = 0 (no queue)

         ── Field 7: DATA_OFFSET (0x0096) ────────────────────────────────
 4C      00 96                         ID          = DATA_OFFSET
 4E      00 04                         Length      = 4
 50      00 00 00 00                   Data        = 0 (no resume offset)
```

The client uses `DATA_REFNUM = 0x0000ABCD` to initiate the HTXF transfer on port + 1.

---

## Example: HTXF Handshake (24-byte)

Following the Download File reply above, the client opens a TCP connection to the transfer port and sends a 24-byte handshake with `HTXF_FLAG_LARGE_FILE | HTXF_FLAG_SIZE64`:

```
 Offset  Bytes                         Field
 ──────  ────────────────────────────  ─────────────────────────────────────
 00      48 54 58 46                   Protocol    = "HTXF"
 04      00 00 AB CD                   RefNum      = 0x0000ABCD (from server reply)
 08      00 00 00 00                   DataSize    = 0 (zeroed; real size in 64-bit field)
 0C      00 00 00 03                   Flags       = HTXF_FLAG_LARGE_FILE | HTXF_FLAG_SIZE64
 10      00 00 00 01 40 00 00 66       DataSize64  = 5,368,709,222 (FFO total)
```

Total handshake: **24 bytes**.

**Field notes:**

- `DataSize` (bytes 8–11) is zero because the transfer exceeds 32 bits. This prevents a legacy server from mistakenly treating it as a 0-byte transfer; a compliant server reads `DataSize64` instead.
- `Flags = 0x00000003` sets both `HTXF_FLAG_LARGE_FILE` (bit 0) and `HTXF_FLAG_SIZE64` (bit 1).
- The server matches `RefNum` to the pending transfer created by the Download File reply, clears the handshake timeout, and begins streaming the FFO.

**Legacy (16-byte) comparison:** For a small file (< 4 GiB) without large-file mode, the handshake would be:

```
 Offset  Bytes                         Field
 ──────  ────────────────────────────  ─────────────────────────────────────
 00      48 54 58 46                   Protocol    = "HTXF"
 04      00 00 AB CD                   RefNum      = 0x0000ABCD
 08      00 12 D6 80                   DataSize    = 1,234,560 (exact 32-bit transfer size)
 0C      00 00 00 00                   Flags       = 0 (no flags)
```

Total handshake: **16 bytes**.

---

Status: draft; subject to refinement.
