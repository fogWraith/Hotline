# Text Encoding Extension

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
- [Known Client Encodings](#known-client-encodings)
- [Configuration](#configuration)
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

The following string data is subject to encoding conversion:

| Data Type | Field IDs | Direction | Notes |
|---|---|---|---|
| **Chat messages** | `DATA_STRING` (101) | Bidirectional | Public chat and private chat rooms |
| **User nicknames** | `DATA_NICK` (102) | Bidirectional | Display name visible to all users |
| **File/folder names** | `DATA_FILENAME` (201) | Bidirectional | Already converted Mac Roman ↔ UTF-8 for filesystem paths |
| **File/folder comments** | `DATA_FILECOMMENT` (210) | Bidirectional | Stored in file metadata |
| **Message board** | `DATA_STRING` (101) | Bidirectional | Flat news postings |
| **Threaded news** | `DATA_NEWSARTDATA` (333) | Bidirectional | News article body text |
| **News titles** | `DATA_NEWSARTTITLE` (328) | Bidirectional | News article titles |
| **News poster** | `DATA_NEWSARTPOSTER` (329) | Read-only | Author name on news articles |
| **Private messages** | `DATA_STRING` (101) | Bidirectional | Direct messages between users |
| **Auto-reply** | `DATA_AUTOREPLY` (215) | Bidirectional | Automatic response text |
| **Chat subjects** | `DATA_CHATSUBJECT` (115) | Bidirectional | Private chat room subject lines |
| **Server name** | `DATA_SERVERNAME` (162) | Server → Client | Server name in login reply |
| **Server agreement** | `DATA_AGREEMENT` (150) | Server → Client | Agreement text |
| **File type/creator strings** | `DATA_FILETYPESTR` (205), `DATA_FILECREATORSTR` (206) | Server → Client | Human-readable type descriptions |

Binary fields (passwords, access bitmasks, icon IDs, transfer data) are **not** affected.

## Server-Side Encoding Detection

The server should determines each client's text encoding using a three-tier priority system:

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

## Known Client Encodings

A reference table of known Hotline clients and their native encodings:

| Client | Version (decimal) | Native Encoding | Notes |
|---|---|---|---|
| Hotline Client 1.2.3 | (none) | Mac Roman | No FieldVersion sent |
| Hotline Client 1.5–1.8 | 150–185 | Mac Roman | Standard classic Mac client |
| Hotline Client 1.9 | 185+ | Mac Roman | Last official release |
| Klein | 199 | UTF-8 | Modern client; will advertise CAPABILITY_TEXT_ENCODING |

Servers may maintain their own lookup tables and extend this list as new clients emerge.

## Implementation Notes

- **Round-trip safety:** When a legacy Mac Roman client uploads a file with a Mac Roman filename, the server decodes it to UTF-8 for storage. When the same client later requests the file list, the server encodes the UTF-8 name back to Mac Roman. The filename must survive this round-trip without data loss for characters in the Mac Roman repertoire.

- **Unmappable characters:** When transcoding from UTF-8 to a legacy encoding, characters outside the target encoding's repertoire are replaced with `?` (0x3F). This is a lossy operation but prevents protocol errors. Clients should be aware that filenames or messages containing characters not in their encoding will show replacement characters.

- **Encoding errors:** If inbound bytes are not valid in the client's assumed encoding, the server should apply best-effort decoding (replacing invalid sequences with the Unicode replacement character U+FFFD) rather than rejecting the transaction.

- **Mixed-encoding servers:** On servers where both Mac Roman and Shift-JIS clients are expected, administrators should be aware that the `DefaultTextEncoding` applies uniformly to all non-negotiating clients. There is no per-user encoding override for legacy clients. HOPE identification, when implemented, could improve this situation.

- **Performance:** Encoding conversion is a per-field operation at transaction time. For typical Hotline message sizes (< 64 KB), the overhead is negligible.
