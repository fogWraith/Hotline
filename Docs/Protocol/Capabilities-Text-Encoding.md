# Text Encoding Extension

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document describes the text encoding extension to the Hotline protocol. It allows clients and servers to negotiate UTF-8 text encoding for all string data, replacing the legacy Mac Roman assumption while remaining backwards compatible with classic Macintosh clients.

For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md).

## Table of Contents

- [Background](#background)
- [The Problem](#the-problem)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
- [Capability Bit](#capability-bit)
- [Affected Data](#affected-data)
- [Server-Side Encoding Detection](#server-side-encoding-detection)
  - [Capability Negotiation (Modern Clients)](#capability-negotiation-modern-clients)
  - [Version Sniffing (Legacy Clients)](#version-sniffing-legacy-clients)
  - [Server Default Fallback](#server-default-fallback)
- [Server-Side Transcoding](#server-side-transcoding)
  - [Inbound (Client → Server)](#inbound-client--server)
  - [Outbound (Server → Client)](#outbound-server--client)
  - [Embedded Binary Fields](#embedded-binary-fields)
  - [Line Ending Normalization](#line-ending-normalization)
- [Client-Side Implementation](#client-side-implementation)
- [Known Client Encodings](#known-client-encodings)
- [Server Configuration](#server-configuration)
- [Implementation Notes](#implementation-notes)

---

## Background

The original Hotline protocol was designed for Classic Mac OS, which used Mac Roman (also known as Macintosh encoding) for all text. There was no encoding negotiation — both sides simply assumed Mac Roman.

Over time, some clients and servers began using other encodings:

- **Shift-JIS** — Japanese
- **Latin-1 (ISO 8859-1)** — Western European
- **UTF-8** — Modern reimplementations

The protocol provides no mechanism for either side to declare which encoding it uses, leading to mojibake (garbled text) when clients with different encodings connect to the same server.

## The Problem

Consider a server with three connected clients:

1. A Classic Mac OS client sending Mac Roman (`ƒ` = `0xC4`)
2. A Japanese client sending Shift-JIS (`ベスト` = `0xCD DE BD C4`)
3. A modern client sending UTF-8 (`ƒ` = `0xC6 0x92`)

Without encoding awareness, the server passes raw bytes between clients. Each client interprets every other client's text using its own encoding, producing garbage. Filenames written to the filesystem suffer the same fate — Mac Roman bytes stored as-is on a UTF-8 filesystem produce corrupted names.

## Compatibility and Negotiation

- UTF-8 mode is activated when a client advertises `CAPABILITY_TEXT_ENCODING` (bit 1 of `DATA_CAPABILITIES`) during login, and the server confirms it in the reply.
- Clients that do not advertise the capability are assumed to speak the server's default text-encoding (typically Mac Roman).
- The capability bit signals **UTF-8 specifically** — not an arbitrary encoding. The bit means "I speak UTF-8; send me UTF-8 and I will send you UTF-8."
- Legacy clients require no changes. The server should transcode between their assumed encoding and its internal UTF-8 storage transparently.

## Capability Bit

| Bit | Mask | Name | Description |
|---|---|---|---|
| 1 | `0x0002` | `CAPABILITY_TEXT_ENCODING` | Client supports UTF-8 for all string data |

This bit is defined in the `DATA_CAPABILITIES` bitmask (field `0x01F0`). See [DATA_CAPABILITIES](Capabilities.md) for the general negotiation flow.

When both `CAPABILITY_LARGE_FILES` (bit 0) and `CAPABILITY_TEXT_ENCODING` (bit 1) are active, the capability bitmask is `0x0003`.

## Affected Data

The following transaction fields carry human-readable text that MUST be transcoded between the client's negotiated encoding and the server's internal UTF-8 representation:

| ID (hex) | Decimal | Name | Direction | Used For |
|---|---|---|---|---|
| `0x0064` | 100 | `DATA_ERROR` | Server → Client | Error messages in reply transactions |
| `0x0065` | 101 | `DATA_STRING` | Bidirectional | Chat messages, private messages, message board posts, server agreement text |
| `0x0066` | 102 | `DATA_NICK` | Bidirectional | User display name |
| `0x0073` | 115 | `DATA_CHATSUBJECT` | Bidirectional | Private chat room subject lines |
| `0x00A2` | 162 | `DATA_SERVERNAME` | Server → Client | Server name in login reply |
| `0x00C9` | 201 | `DATA_FILENAME` | Bidirectional | File/folder names |
| `0x00CD` | 205 | `DATA_FILETYPESTR` | Server → Client | Human-readable file type description |
| `0x00CE` | 206 | `DATA_FILECREATORSTR` | Server → Client | Human-readable file creator description |
| `0x00D2` | 210 | `DATA_FILECOMMENT` | Bidirectional | File/folder comments |
| `0x00D3` | 211 | `DATA_FILENEWNAME` | Client → Server | New filename for rename operations |
| `0x00D6` | 214 | `DATA_QUOTINGMSG` | Bidirectional | Quoted message text in private messages |
| `0x00D7` | 215 | `DATA_AUTOREPLY` | Bidirectional | Automatic response text |
| `0x0142` | 322 | `DATA_NEWSCATNAME` | Bidirectional | News category names |
| `0x0148` | 328 | `DATA_NEWSARTTITLE` | Bidirectional | News article titles |
| `0x0149` | 329 | `DATA_NEWSARTPOSTER` | Read-only | Author name on news articles |
| `0x014D` | 333 | `DATA_NEWSARTDATA` | Bidirectional | News article body text |

**Note on server agreement:** The agreement text displayed by `TranShowAgreement (109)` is carried in `DATA_STRING (101)`, not in `DATA_SERVERAGREEMENT (150)`. Although field ID 150 exists in the protocol, Hotline uses `DATA_STRING` as the carrier for agreement text. Servers MUST transcode this field like any other `DATA_STRING` instance.

Binary fields (passwords, access bitmasks, icon IDs, transfer data) are **not** affected. Any field not in the table above SHOULD be treated as opaque binary and passed through without conversion.

## Server-Side Encoding Detection

The server SHOULD determine each client's text encoding using a three-tier priority system:

### Capability Negotiation (Modern Clients)

**Priority: Highest**

If the client sends `CAPABILITY_TEXT_ENCODING` in `DATA_CAPABILITIES` during login, the server confirms `utf-8` for the session by echoing the bit back. All string data exchanged with this client uses UTF-8.

### Version Sniffing (Legacy Clients)

**Priority: Medium**

Legacy clients do not send `DATA_CAPABILITIES`. The server examines the client's `DATA_VERSION` field (160) — a 2-byte unsigned integer — to make a best-effort guess:

| Version | Decimal | Likely Client | Assumed Encoding |
|---|---|---|---|
| absent | — | Hotline 1.2.3 | Server default |
| `0x0096` | 150 | Hotline 1.5.x | Server default |
| `0x00B5` | 181 | Hotline 1.8.x | Server default |
| `0x00B9` | 185 | Hotline 1.8.5 | Server default |

Without HOPE `app_id` identification (see [HOPE Secure Login](HOPE-Secure-Login.md)), the version number alone cannot distinguish between Mac Roman, Shift-JIS, and Latin-1 clients — all classic Mac clients send similar version numbers regardless of system encoding. Version sniffing therefore falls back to the server default for all legacy clients.

On that note, third-party clients (clones) either set a unique version number, or sent the same as the official Hotline Connect Client, which in turn makes version sniffing unreliable at best.

Future integration with HOPE `app_id` and `app_string` will enable more precise identification of specific clients and their native encodings.

### Server Default Fallback

**Priority: Lowest**

The server configuration should provide a default text-encoding — the encoding assumed for any client that does not negotiate via capability or match a known version signature.

For servers catering primarily to classic Mac clients, this should be `macintosh` (Mac Roman). For servers in a Japanese community, `shift-jis`. For modern-only servers, `utf-8`.

## Server-Side Transcoding

The server should act as an encoding bridge between clients. Internally, all text is stored as UTF-8.

### Inbound (Client → Server)

```
Client bytes → Decode from client's encoding → UTF-8 string → Store/process
```

When the server receives string data from a client, it decodes the bytes from the client's negotiated encoding into UTF-8 before storing or forwarding.

### Outbound (Server → Client)

```
UTF-8 string → Encode to client's encoding → Client bytes → Send
```

When the server sends string data to a client, it encodes the internal UTF-8 representation into the client's negotiated encoding. Characters that cannot be represented in the target encoding are replaced with `?` (0x3F).

### Embedded Binary Fields

Some binary fields contain embedded text strings at fixed offsets. These fields are NOT simple text and MUST NOT be decoded wholesale. Instead, implementations MUST parse the binary structure, extract the embedded string portion, transcode it, and reassemble the field with updated length headers.

| ID (hex) | Decimal | Name | Embedded Text | Layout |
|---|---|---|---|---|
| `0x00C8` | 200 | `DATA_FILE_NAME_WITH_INFO` | Filename at offset 20 | `Type[4] Creator[4] FileSize[4] RSVD[4] NameScript[2] NameSize[2] Name[variable]` |
| `0x012C` | 300 | `DATA_USERNAME_WITH_INFO` | Username at offset 8 | `UserID[2] IconID[2] Flags[2] NameLen[2] Name[variable] [Color[4] optional]` |

For `DATA_FILE_NAME_WITH_INFO`: read `NameSize` from bytes 18–19, extract `Name` at offset 20 with that length, transcode, then write back with an updated `NameSize`.

For `DATA_USERNAME_WITH_INFO`: read `NameLen` from bytes 6–7, extract `Name` at offset 8 with that length, transcode, then write back with an updated `NameLen`. Preserve any trailing data (e.g. a 4-byte color extension) after the name.

### Line Ending Normalization

Classic Macintosh clients use CR (`0x0D`) as the line separator; modern systems use LF (`0x0A`). To maintain compatibility:

- **Inbound (client → server):** After decoding to UTF-8, replace CR (`0x0D`) with LF (`0x0A`) for internal storage and forwarding to UTF-8 clients.
- **Outbound (server → client):** Before encoding to a legacy encoding, replace LF (`0x0A`) with CR (`0x0D`) for Mac Roman, Shift-JIS, and Latin-1 clients. UTF-8 clients receive LF as-is.

This ensures chat messages and news articles display correctly regardless of the client's line ending convention.

## Client-Side Implementation

While the server performs the actual transcoding, modern clients MUST also implement their side of the negotiation:

1. **Advertise the capability:** Include `CAPABILITY_TEXT_ENCODING` (bit 1) in `DATA_CAPABILITIES` during login. Clients supporting multiple extensions combine the bits (e.g. `0x0003` for large-file + text-encoding).

2. **Check the server's reply:** Parse `DATA_CAPABILITIES` from the login reply. If bit 1 is set, the server confirmed UTF-8 mode — all subsequent text fields will be UTF-8. Store this as a per-session flag.

3. **Fallback to Mac Roman:** If the server does not echo the capability (field absent or bit 1 clear), the client MUST encode outgoing text as Mac Roman and decode incoming text from Mac Roman. This ensures compatibility with legacy servers that do not support the extension.

4. **Encoding functions:** Implement a pair of functions:
   - **Decode** (wire bytes → internal string): If UTF-8 mode, return bytes as-is. Otherwise, decode from Mac Roman.
   - **Encode** (internal string → wire bytes): If UTF-8 mode, return UTF-8 bytes. Otherwise, encode to Mac Roman.

5. **Apply to all text fields:** Use the encode/decode functions for every text field listed in the Affected Data table. Do not selectively skip fields.

## Known Client Encodings

A reference table of known Hotline clients and their native encodings:

| Client | Version (decimal) | Native Encoding | Notes |
|---|---|---|---|
| Hotline Client 1.2.3 | (none) | Mac Roman | No FieldVersion sent |
| Hotline Client 1.5–1.8 | 150–185 | Mac Roman | Standard classic Mac client |
| Hotline Client 1.9 | 185+ | Mac Roman | Last official release |
| Klein | 199 | UTF-8 | Modern client; will advertise CAPABILITY_TEXT_ENCODING |

Servers may maintain their own lookup tables and extend this list as new clients emerge.

## Server Configuration

Servers SHOULD expose the following configuration options:

| Setting | Type | Default | Description |
|---|---|---|---|
| `DefaultTextEncoding` | String | `"macintosh"` | Encoding assumed for clients that do not negotiate via capability. Valid values: `"macintosh"`, `"latin-1"`, `"shift-jis"`, `"utf-8"`. |

Administrators set this based on their expected client population:

- **Classic Mac community:** `"macintosh"` (Mac Roman) — the safest default for interoperability with original Hotline clients.
- **Japanese community:** `"shift-jis"` — matches native encoding of Japanese Mac OS and Windows Hotline clients.
- **Modern-only server:** `"utf-8"` — appropriate when all clients are known to be modern implementations.

There is no per-user encoding override for legacy clients. All non-negotiating clients share the server default. HOPE `app_id` identification, when implemented, could enable per-client encoding selection.

## Implementation Notes

- **Internal storage:** Servers MUST store all text internally as UTF-8 (the canonical encoding). Inbound text from legacy clients is decoded to UTF-8 on receipt; outbound text is encoded to the target client's encoding on send. This means the transcoding is a per-field, per-transaction operation at I/O time — the server's internal data structures, database, and filesystem all use UTF-8.

- **Per-connection state:** The negotiated encoding MUST be stored per client connection. Each connected client may have a different encoding. Servers MUST NOT use a single global encoding for all clients.

- **Transcoding scope:** Transcoding applies to the transaction field level. For each inbound or outbound transaction, iterate through the field list. If a field's type matches an entry in the Affected Data table, apply the encoding conversion. For embedded binary fields (`DATA_FILE_NAME_WITH_INFO`, `DATA_USERNAME_WITH_INFO`), apply the special parsing described in "Embedded Binary Fields". All other fields pass through unmodified.

- **Round-trip safety:** When a legacy Mac Roman client uploads a file with a Mac Roman filename, the server decodes it to UTF-8 for storage. When the same client later requests the file list, the server encodes the UTF-8 name back to Mac Roman. The filename MUST survive this round-trip without data loss for characters in the Mac Roman repertoire.

- **Unmappable characters:** When transcoding from UTF-8 to a legacy encoding, characters outside the target encoding's repertoire MUST be replaced with `?` (`0x3F`). This is a lossy operation but prevents protocol errors. Clients should be aware that filenames or messages containing characters not in their encoding will show replacement characters.

- **Encoding errors:** If inbound bytes are not valid in the client's assumed encoding, the server SHOULD apply best-effort decoding (replacing invalid sequences with the Unicode replacement character U+FFFD) rather than rejecting the transaction. Never drop a transaction due to encoding errors.

- **Mixed-encoding servers:** On servers where both Mac Roman and Shift-JIS clients are expected, administrators should be aware that the `DefaultTextEncoding` applies uniformly to all non-negotiating clients. There is no per-user encoding override for legacy clients. HOPE identification, when implemented, could improve this situation.

- **Performance:** Encoding conversion is a per-field operation at transaction time. For typical Hotline message sizes (< 64 KB), the overhead is negligible.

- **Encoding libraries:** Most languages provide encoding conversion libraries. Recommended references:
  - Mac Roman: IANA name `macintosh`, Windows codepage 10000
  - ISO 8859-1 (Latin-1): IANA name `ISO-8859-1`
  - Shift-JIS: IANA name `Shift_JIS`, Windows codepage 932
  - UTF-8 is native in most modern languages (Go, Rust, Python 3, JavaScript, Swift)

---

Status: draft; subject to refinement.
