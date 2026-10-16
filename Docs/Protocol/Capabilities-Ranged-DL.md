# Ranged Download Extension

> Last updated: October 16, 2026
>
> **Status:** Implemented by the reference server, Janus, from 2.0.26.

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This extension lets a client download a byte range of a file's data fork: a given number of bytes from a given offset, sent as they are and nothing more. The server sends exactly the range and ends the transfer normally.

It works with or without the [Large File Extension](Capabilities-Large-File.md). Without it, offsets and lengths are 32-bit, as everywhere else in the protocol; with it, they may be 64-bit. For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md).

## Table of Contents

- [Motivation](#motivation)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
- [Fields](#fields)
- [Download File (202) with a Range](#download-file-202-with-a-range)
- [The Transfer](#the-transfer)
- [Server Behaviour](#server-behaviour)
- [Client Behaviour](#client-behaviour)
- [Examples](#examples)

---

## Motivation

A client that reads only part of a file has no way to say so. Download File (202) can resume from an offset, and the preview mode (`DATA_FILE_TRANSFER_OPTIONS` = 2) sends the bare data fork from its start, but neither takes a length: the server sends to the end of the file.

A client that views archives or disk images on the server reads them this way: the table of contents first, then each file or page as it is wanted, without downloading the whole archive. Today it can only ask to resume at the piece it needs, read that many bytes, and hang up. That works with every server, but:

- **The server sends more than is wanted.** Until the client's reset reaches it, the server keeps writing. A server's own count of bytes sent overstates the waste, since much of it is still in the server's send buffer when the reset arrives and never crosses the network; the real overshoot per read is roughly one TCP window.
- **The server sees a broken transfer.** Each read ends in a connection reset or broken pipe, which a server may log as a failed download, though nothing went wrong.
- **The client reads past headers it doesn't need.** A resumed download sends a flattened file object (FFO) header and the fork headers before the data.

With a range, the server sends exactly what was asked for and closes the transfer as complete.

**What it saves.** The gain is mostly cleanliness: complete transfers instead of broken ones, and no headers to read past. Bandwidth saved is real but modest, and depends on the reading pattern:

- **Many small reads** gain most. Walking the headers of a RAR archive can take dozens of reads; each wastes up to a window of overshoot, a few MB for one archive opened.
- **Large reads** gain little.
- **Streamed media** gains nothing: a player reading a video in order wants the bytes that follow.
- **Whole-file downloads** are unaffected; this extension is only for reading part of a data fork.

## Compatibility and Negotiation

- A new capability bit, `CAPABILITY_RANGED_DOWNLOAD` (bit 12, mask `0x1000`), in `DATA_CAPABILITIES`. A client advertises it at login; a server that supports ranges confirms it.
- A client MUST NOT send a range unless the server confirmed the bit. Without it, the client falls back to resuming and hanging up.
- The bit does not depend on `CAPABILITY_LARGE_FILES` (bit 0). A client MAY advertise, and a server MAY confirm, bit 12 without bit 0. The session is then a **32-bit session** for ranges; when both bits are confirmed, it is a **64-bit session**.
- A server that confirms bit 12 MUST accept 32-bit ranges in either kind of session.

## Fields

| ID | Decimal | Name | Type |
|---|---|---|---|
| `0x01FC` | 508 | `DATA_RANGE_OFFSET` | Unsigned integer, big-endian, 2, 4 or 8 bytes |
| `0x01FD` | 509 | `DATA_RANGE_LENGTH` | Unsigned integer, big-endian, 2, 4 or 8 bytes |

Both are integers of variable width, as Hotline integer fields already are; the field's data length gives the width:

- **2 or 4 bytes** is always allowed. The original client sends an integer that fits in 2 bytes in 2, and a client may build these fields with the same code.
- **8 bytes** is allowed only in a 64-bit session. A sender SHOULD use 2 or 4 bytes whenever the value fits, so that the same request is valid in either kind of session.
- A server MUST reply with an error to a field of any other width, or to an 8-byte field in a 32-bit session.

## Download File (202) with a Range

A ranged download is a preview with an offset and a length.

**Request**, beside the usual file name (201) and file path (202):

| Field | Value |
|---|---|
| `DATA_FILE_TRANSFER_OPTIONS` (204) | `2`: the bare data fork, as a preview |
| `DATA_RANGE_OFFSET` | Where the range starts in the data fork. `0` if absent. |
| `DATA_RANGE_LENGTH` | How many bytes are wanted from the offset. `0` means to the end of the fork. |

A range may be asked of any file. The original client sent option `2` only for text and picture files (TEXT, JPEG, GIFF, BMP, PICT), but that was the client's choice; a server MUST NOT refuse a range because of a file's type or creator.

The presence of `DATA_RANGE_LENGTH` is what makes the request a range. Without it, the request is an ordinary preview, and the server MUST ignore `DATA_RANGE_OFFSET`.

A range request MUST NOT carry `DATA_FILE_RESUME_DATA` (203, RFLT) or `DATA_OFFSET64` (`0x01F2`); its offset is `DATA_RANGE_OFFSET`. A server receiving either with a range MUST reply with an error.

**Reply:**

| Field | 32-bit session | 64-bit session |
|---|---|---|
| Reference number (107) | As for any download | As for any download |
| Waiting count (116) | As for any download: `0` unless the server queues this range | The same |
| `DATA_TRANSFERSIZE` (108) | The number of bytes the transfer will carry: the range, cut short at the end of the fork | The same, clamped to `0xFFFFFFFF` |
| `DATA_FILESIZE` (207) | The data fork's full size | The same, clamped to `0xFFFFFFFF` |
| `DATA_XFERSIZE64` (`0x01F3`) | Not sent | The number of bytes the transfer will carry |
| `DATA_FILESIZE64` (`0x01F1`) | Not sent | The data fork's full size |

The fork's full size is sent so the client knows the file's length whatever range it asked for; an archive reader needs it to find a table of contents kept at the end.

**Errors.** The server replies with an ordinary error reply (a non-zero error code, with a message in `DATA_ERROR` (100) for the user), and offers no transfer, when:

- the offset is past the end of the fork. (An offset exactly at the end is not an error: whatever the length, the range is cut short there and is empty. A client reading an empty file from its start gets an empty transfer, as it would from a preview.)
- in a 32-bit session, the data fork is larger than `0xFFFFFFFF` bytes. Such a file is not listed for a 32-bit client in the first place (see the Large File Extension's rules for Get File Name List), so this only happens to a client asking for a name it was not given;
- the request breaks the rules above (a field's width, RFLT or `DATA_OFFSET64` alongside a range).

Everything Download File already refuses is refused for a range in the same way: a name that does not exist or is a folder, a path the user may not download from, missing privileges.

## The Transfer

The client opens the transfer connection as for any download and sends the HTXF handshake with the reference number:

- **In a 32-bit session**, the classic 16-byte handshake.
- **In a 64-bit session**, the handshake of the Large File Extension: its large-file flag, and its 64-bit length when the range exceeds `0xFFFFFFFF` bytes.

Whatever protects the session's other transfers (TLS on the transfer port, or HOPE transfer encryption) protects a range in the same way; this extension changes only what is sent, not how.

The server then sends exactly the number of bytes given in the reply's transfer size, from the offset, with no FFO and no fork headers. A transfer size of `0` sends nothing after the handshake.

The client reads until it has that many bytes and then closes the connection. The server, having sent the range, ends the transfer as after any complete download; a server that waits for the client to close first (as some do, for the sake of older clients) is not kept waiting.

## Server Behaviour

- **Privileges** are those of Download File: a range is a download. Drop boxes and other places a user may not download from refuse ranges the same way.
- **Throttling and transfer limits** apply as to any download. A server MAY treat a range below some size as not counting toward its download limits or queue, since a viewer may read several pieces at once and should not wait behind large downloads. A server doing so SHOULD cap that size and how many such ranges one client may have at once, so that ranges cannot be used to fetch a whole file past the queue. A range that does queue is reported like any queued download (Download Info, 211).
- **The download count** (in a file's info or catalogue) SHOULD NOT be incremented for a range: a range is a look, not a download.
- **Logging:** a range that ends after sending its length is a complete download. A client that hangs up before then has stopped the download, which is not an error.
- **The file may change** between two ranges. The server serves each range from the file as it is when the request arrives, and does not promise that two ranges come from the same version. A client that cares compares the fork's full size between replies.

## Client Behaviour

- Ask for what is needed, and no more. Ranges are cheap; a client reading a large archive SHOULD ask for each piece rather than a run of the file "to be safe".
- Use the reply's transfer size, not the length asked for: it is less when the range runs past the end of the fork, and `0` at the end. A client that needs exactly the length it asked for treats a shorter range as the file having changed.
- A client MAY have several ranges open at once, subject to the server's limits. Each is its own Download File and its own transfer connection.
- A non-zero waiting count means the server has queued the range; the client handles it as for any queued download.
- Reading a range to its end and closing is the normal ending. Hanging up early is still allowed (a user closing the viewer), and is not an error.
- In a 32-bit session, never ask for a range that ends past `0xFFFFFFFF`.
- Without the capability, fall back to resuming and hanging up, reading past the FFO and fork headers.

## Examples

Field data only (ID, length, data); the transaction header and the other fields of a Download File request are as usual.

**32-bit session:** 65,536 bytes from offset 1,048,576.

```
00 CC  00 02  00 02                     DATA_FILE_TRANSFER_OPTIONS = 2
01 FC  00 04  00 10 00 00               DATA_RANGE_OFFSET = 1048576
01 FD  00 04  00 01 00 00               DATA_RANGE_LENGTH = 65536
```

A value that fits in 2 bytes may be sent in 2: the first 22 bytes would be `01 FD  00 02  00 16`, with no offset field at all.

The reply carries `DATA_TRANSFERSIZE` = 65,536 (fewer if the fork ends sooner) and `DATA_FILESIZE` = the fork's size. The transfer is the classic 16-byte handshake, then 65,536 bytes.

**64-bit session:** the last 22 bytes of a 6 GiB file (a ZIP archive's end record), from offset 6,442,450,922, which does not fit in 4 bytes.

```
00 CC  00 02  00 02                     DATA_FILE_TRANSFER_OPTIONS = 2
01 FC  00 08  00 00 00 01 7F FF FF EA   DATA_RANGE_OFFSET = 6442450922
01 FD  00 04  00 00 00 16               DATA_RANGE_LENGTH = 22
```

The length fits in 4 bytes, so it is sent in 4. The reply carries `DATA_TRANSFERSIZE` = 22, `DATA_XFERSIZE64` = 22, `DATA_FILESIZE` = `0xFFFFFFFF` (clamped) and `DATA_FILESIZE64` = 6,442,450,944.
