# Inline Media Extension

This document describes the inline media extension to the Hotline protocol. It enables clients to attach images to chat messages, which capable clients render inline. The extension is designed to be safe by default: image bytes are not embedded directly in chat transactions, and servers always validate, canonicalise, and re-encode the bytes before relaying.

For the general capability negotiation mechanism, see [Capabilities.md](Capabilities.md).

## Table of Contents

- [Scope](#scope)
- [Design Goals](#design-goals)
- [Terminology](#terminology)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
- [New Data Objects](#new-data-objects)
- [New Transactions](#new-transactions)
  - [Upload Media (TranUploadMedia)](#upload-media-tranuploadmedia)
  - [Download Media (TranDownloadMedia)](#download-media-trandownloadmedia)
- [Chat Messages with Media](#chat-messages-with-media)
  - [Public Chat (105 / 106)](#public-chat-105--106)
  - [Private Messages (108 / 104)](#private-messages-108--104)
  - [Private Chat Rooms (105 / 106 with ChatID)](#private-chat-rooms-105--106-with-chatid)
- [Server Behaviour](#server-behaviour)
  - [Upload Validation Pipeline](#upload-validation-pipeline)
  - [Authorization](#authorization)
  - [Relaying to Capable Clients](#relaying-to-capable-clients)
  - [Legacy Client Fallback](#legacy-client-fallback)
  - [Media Gateway (optional, public contexts only)](#media-gateway-optional-public-contexts-only)
  - [Resource Limits](#resource-limits)
  - [Logging and Privacy](#logging-and-privacy)
  - [Storage and Lifetime](#storage-and-lifetime)
- [Client Behaviour](#client-behaviour)
  - [Sending Media](#sending-media)
  - [Receiving Media](#receiving-media)
  - [Supported Formats](#supported-formats)
- [Security Considerations](#security-considerations)
- [Implementation Notes](#implementation-notes)
- [Capability Bit](#capability-bit)

---

## Scope

Adds the ability to attach a single image to a chat message (public chat, private message, or private chat room). The image bytes are not carried inside the chat transaction; instead, the sender first uploads the bytes to the server using a dedicated transaction and receives an opaque media handle. The chat transaction then carries only the handle and a MIME hint. Capable recipients fetch the canonicalised bytes from the server using a second dedicated transaction.

This separation keeps chat transactions small, gives the server a clean place to validate and re-encode bytes, and naturally enforces authorization (only clients that received the chat message can resolve the handle).

## Design Goals

- **Safe by default.** Servers MUST validate, canonicalise, and re-encode media before any other client sees it.
- **Bounded.** No single chat transaction grows beyond standard Hotline field-size constraints.
- **Authorized.** Possessing a media handle does not by itself grant access to the bytes; the server checks that the requester was a recipient of the originating chat message.
- **Backwards compatible.** Legacy clients that do not negotiate the capability are unaffected; they continue to receive plain text.
- **Privacy preserving.** Private messages and private chat rooms never leak media to public third-party hosts.

## Terminology

- *Inline media*: An image attached to a chat message and rendered within the chat view.
- *Media handle*: An opaque, server-issued, unguessable token that identifies a single uploaded image. Treated by clients as a string of bytes; format is server-defined.
- *Capable client*: A client that has negotiated `CAPABILITY_INLINE_MEDIA` for the session.
- *Legacy client*: A client that has not negotiated the capability.
- *Media gateway*: An optional external HTTP service used by the server, exclusively for the public-chat legacy fallback, to provide a clickable URL to legacy clients.

## Compatibility and Negotiation

- Inline media is activated when a client advertises `CAPABILITY_INLINE_MEDIA` (bit 3 of `DATA_CAPABILITIES`) during login, and the server confirms it in the reply.
- A server that has not enabled this extension MUST NOT confirm the bit. Clients MUST NOT send media-related transactions or fields if the server did not confirm the bit.
- A client that did not negotiate the capability MUST NOT send `DATA_CHAT_MEDIA_ID`, `DATA_CHAT_MEDIA_TYPE`, or the new transactions. Servers MUST drop these fields from any inbound transaction whose sender did not negotiate the capability, and MUST NOT relay them.
- Servers MUST strip media fields from any relayed transaction whose recipient did not negotiate the capability.
- Servers that do not implement this extension may reject the new transactions as unknown. Clients MUST handle that case by suppressing the media UI for the session.

## New Data Objects

| ID (hex) | Name | Type | Notes |
|---|---|---|---|
| `0x0202` | `DATA_CHAT_MEDIA_ID` | Binary, ≤ 64 bytes | Opaque server-issued media handle. Carried in chat transactions. |
| `0x0201` | `DATA_CHAT_MEDIA_TYPE` | String, ≤ 64 bytes | Canonical MIME type of the image after server-side re-encoding. The sender's declared type is a hint only; the server overwrites this field with the canonical value before relaying. |
| `0x0203` | `DATA_CHAT_MEDIA_PAYLOAD` | Binary | Image bytes. Used **only** in `TranUploadMedia` (request) and `TranDownloadMedia` (reply). Never appears in chat transactions. |
| `0x0204` | `DATA_CHAT_MEDIA_DECLARED_TYPE` | String, ≤ 64 bytes | Sender-declared MIME type. Used **only** in `TranUploadMedia`; treated as a hint. |
| `0x0205` | `DATA_CHAT_MEDIA_WIDTH` | Unsigned 32-bit | Canonical image width in pixels. Set by the server in chat transactions and in the download reply. |
| `0x0206` | `DATA_CHAT_MEDIA_HEIGHT` | Unsigned 32-bit | Canonical image height in pixels. Set by the server. |
| `0x0207` | `DATA_CHAT_MEDIA_BYTES` | Unsigned 32-bit | Canonical image size in bytes. Set by the server. |
| `0x0208` | `DATA_CHAT_MEDIA_UPLOAD_TOKEN` | Binary, ≤ 64 bytes | Server-issued upload session token for chunked uploads. |
| `0x0209` | `DATA_CHAT_MEDIA_PART_INDEX` | Unsigned 16-bit | Zero-based chunk index. |
| `0x020A` | `DATA_CHAT_MEDIA_PART_COUNT` | Unsigned 16-bit | Total chunk count. |
| `0x020B` | `DATA_CHAT_MEDIA_PART_FINAL` | Unsigned 8-bit | Non-zero on the final chunk. |

`DATA_CHAT_MEDIA_ID` and `DATA_CHAT_MEDIA_TYPE` are companion fields. In any chat transaction either both MUST be present or both MUST be absent. Receivers MUST reject a transaction that contains exactly one of the two.

`DATA_CHAT_MEDIA_WIDTH`, `DATA_CHAT_MEDIA_HEIGHT`, and `DATA_CHAT_MEDIA_BYTES` are server-supplied hints to help clients allocate UI space before fetching. They are advisory; clients MUST NOT trust them as substitutes for validating the bytes returned by `TranDownloadMedia`.

## New Transactions

### Upload Media (TranUploadMedia)

**Type:** `TranUploadMedia` = `750` (`0x02EE`).

**Direction:** Client → Server.

**Purpose:** Submit image bytes to the server. On success, the server returns an opaque media handle that can be referenced in a subsequent chat transaction.

**Single-shot request fields (when bytes fit in one field):**

| Field ID | Name | Required | Notes |
|---|---|---|---|
| `0x0203` | `DATA_CHAT_MEDIA_PAYLOAD` | Yes | Image bytes. |
| `0x0204` | `DATA_CHAT_MEDIA_DECLARED_TYPE` | No | Sender-declared MIME type (hint). |
| `0x020B` | `DATA_CHAT_MEDIA_PART_FINAL` | Yes | Set to a non-zero value to indicate this is the only/final chunk. |

**Reply fields on the final chunk (success):**

| Field ID | Name | Notes |
|---|---|---|
| `0x0202` | `DATA_CHAT_MEDIA_ID` | Server-issued opaque handle. |
| `0x0201` | `DATA_CHAT_MEDIA_TYPE` | Canonical MIME type after re-encoding. |
| `0x0205` | `DATA_CHAT_MEDIA_WIDTH` | Canonical width. |
| `0x0206` | `DATA_CHAT_MEDIA_HEIGHT` | Canonical height. |
| `0x0207` | `DATA_CHAT_MEDIA_BYTES` | Canonical byte size. |

**Reply on failure:** standard error transaction (flags = 1, `DATA_ERROR` with a generic, non-enumerating message such as `"Media rejected"`).

**Chunked upload (when bytes exceed a single field):**

The client sends the upload as a sequence of `TranUploadMedia` transactions. The first transaction carries the first chunk and `DATA_CHAT_MEDIA_PART_COUNT`; the server's reply carries `DATA_CHAT_MEDIA_UPLOAD_TOKEN`. Each subsequent transaction carries the upload token, the next chunk, and the next `DATA_CHAT_MEDIA_PART_INDEX`. The final chunk carries `DATA_CHAT_MEDIA_PART_FINAL` set to a non-zero value; the server's final reply carries the media-handle reply fields above.

| Field ID | Name | First chunk | Subsequent chunks | Notes |
|---|---|---|---|---|
| `0x0203` | `DATA_CHAT_MEDIA_PAYLOAD` | Yes | Yes | This chunk's bytes. |
| `0x0204` | `DATA_CHAT_MEDIA_DECLARED_TYPE` | Optional | MUST NOT | Hint, first chunk only. |
| `0x0208` | `DATA_CHAT_MEDIA_UPLOAD_TOKEN` | MUST NOT | Yes | Echoed from the first reply. |
| `0x0209` | `DATA_CHAT_MEDIA_PART_INDEX` | Optional (= 0) | Yes | Zero-based. |
| `0x020A` | `DATA_CHAT_MEDIA_PART_COUNT` | Yes | MUST NOT | Total chunk count. |
| `0x020B` | `DATA_CHAT_MEDIA_PART_FINAL` | Set if single-shot | Set on final chunk | Non-zero on final chunk. |

The upload token expires after a server-defined idle timeout (recommended 30 seconds). Servers MUST discard partially uploaded data on timeout, on receipt of an out-of-order or duplicate index, or on a chunk-count mismatch, and MUST NOT issue a media handle until all chunks have been received and the full validation pipeline has succeeded on the assembled payload.

Servers MAY refuse chunked uploads entirely and reject any first chunk that declares `DATA_CHAT_MEDIA_PART_COUNT > 1`.

### Download Media (TranDownloadMedia)

**Type:** `TranDownloadMedia` = `751` (`0x02EF`).

**Direction:** Client → Server.

**Purpose:** Retrieve the canonical bytes for a media handle previously announced in a chat transaction.

**Request fields:**

| Field ID | Name | Required | Notes |
|---|---|---|---|
| `0x0202` | `DATA_CHAT_MEDIA_ID` | Yes | Handle from the chat transaction. |
| `0x0209` | `DATA_CHAT_MEDIA_PART_INDEX` | No | Zero-based chunk index when fetching a chunked reply. Absent on the first request. |

**Reply fields:**

| Field ID | Name | Notes |
|---|---|---|
| `0x0203` | `DATA_CHAT_MEDIA_PAYLOAD` | Canonical image bytes (this chunk). |
| `0x0201` | `DATA_CHAT_MEDIA_TYPE` | Canonical MIME type. |
| `0x020A` | `DATA_CHAT_MEDIA_PART_COUNT` | Total chunk count. |
| `0x020B` | `DATA_CHAT_MEDIA_PART_FINAL` | Non-zero on the final chunk. |

Servers MUST authorize each download request: the requester MUST have been one of the recipients of the chat transaction that announced the handle, and the handle MUST still be live. Unauthorized or expired requests MUST return a generic error (`"Media not found"`); servers MUST NOT distinguish "expired" from "not authorized" in the error response.

## Chat Messages with Media

### Public Chat (105 / 106)

**Client → Server (TranChatSend / 105):**

| Field ID | Name | Required | Notes |
|---|---|---|---|
| `0x0065` | `DATA_DATA` | Yes | Text content. May be a fallback such as `[image]`, or a caption. |
| `0x006D` | `DATA_CHATOPTIONS` | No | Standard chat options (0 = normal, 1 = emote). |
| `0x0072` | `DATA_CHATID` | No | Private chat room ID (absent for public chat). |
| `0x0202` | `DATA_CHAT_MEDIA_ID` | No | Media handle returned by an earlier `TranUploadMedia`. |
| `0x0201` | `DATA_CHAT_MEDIA_TYPE` | No | Canonical MIME hint (server overwrites with its own canonical value before relaying). |

**Server → Capable Clients (TranChatMsg / 106):**

| Field ID | Name | Notes |
|---|---|---|
| `0x0065` | `DATA_DATA` | Formatted text. |
| `0x0202` | `DATA_CHAT_MEDIA_ID` | Media handle. |
| `0x0201` | `DATA_CHAT_MEDIA_TYPE` | Canonical MIME type. |
| `0x0205` | `DATA_CHAT_MEDIA_WIDTH` | Canonical width. |
| `0x0206` | `DATA_CHAT_MEDIA_HEIGHT` | Canonical height. |
| `0x0207` | `DATA_CHAT_MEDIA_BYTES` | Canonical byte size. |

**Server → Legacy Clients (TranChatMsg / 106):** all media fields stripped. See [Legacy Client Fallback](#legacy-client-fallback).

### Private Messages (108 / 104)

**Client → Server (TranSendInstantMsg / 108):**

| Field ID | Name | Required | Notes |
|---|---|---|---|
| `0x0065` | `DATA_DATA` | Yes | Message text. |
| `0x0067` | `DATA_UID` | Yes | Recipient user ID. |
| `0x0071` | `DATA_OPTIONS` | No | Message options. |
| `0x0202` | `DATA_CHAT_MEDIA_ID` | No | Media handle. |
| `0x0201` | `DATA_CHAT_MEDIA_TYPE` | No | Canonical MIME hint. |

**Server → Recipient (TranServerMsg / 104):** media fields included only if the recipient negotiated the capability.

### Private Chat Rooms (105 / 106 with ChatID)

Same fields as public chat. Per-recipient capability checks apply. The server MUST verify that the sender is a member of the chat room before relaying.

## Server Behaviour

### Upload Validation Pipeline

For every `TranUploadMedia` request the server MUST perform, in order:

1. **Authorization check.** Reject if the sender lacks `AccessSendMedia` (bit 57; see [Authorization](#authorization)).
2. **Per-account and per-IP quotas.** Reject if the sender has exceeded the configured rate or volume limits.
3. **Encoded size check.** Reject if total payload exceeds the configured maximum (recommended default: 256 KB) once all chunks are assembled.
4. **Magic-byte sniff.** Read the leading bytes of the assembled payload and confirm they match one of the server's allowed formats. Reject if no allowed magic matches. The sender's declared MIME type is a hint only and MUST NOT be used to skip this check.
5. **Trailing-data check.** Confirm there is no significant data after the natural end-of-image marker for the detected format. Reject otherwise.
6. **Dimension probe.** Decode only the image header and read the declared width and height. Reject if `width × height` exceeds the configured pixel cap (recommended default: 2048 × 2048 = ~4.2 megapixels) or if either dimension is below 1 or above 4096.
7. **Bounded full decode.** Decode the full image with a memory and time budget. Reject on decoder error, timeout, or budget exhaustion.
8. **Animated-image checks.** For animated formats, reject if frame count, total decoded bytes, or total animation duration exceed the configured caps.
9. **Re-encode.** Encode the decoded pixels into a canonical output (recommended: PNG for images with an alpha channel, JPEG otherwise; GIF preserved for animated GIF input). The output MUST contain no metadata: no EXIF, no ICC profile, no XMP, no PNG ancillary text chunks, no GIF comment extensions. The bytes that result from this step are what the server stores and relays.
10. **Issue handle.** Generate a media handle with at least 128 bits of cryptographically random entropy. Store the canonical bytes, the canonical MIME type, dimensions, byte size, and the sender's identity, keyed by the handle.

If any step fails, the server MUST return a generic error transaction. The error message MUST NOT reveal which step failed beyond a coarse category (e.g. `"Media too large"`, `"Unsupported media"`, `"Media rejected"`). The server MUST NOT relay the original bytes to any client under any circumstance.

### Authorization

Servers MUST gate sending media on a dedicated permission, **`AccessSendMedia` (privilege bit 57)**. This bit lives in the standard 64-bit `AccessBitmap` carried by `FieldUserAccess` (110); it requires no new field. Operators SHOULD configure this permission off by default for guest accounts.

Clients running on legacy account stores that mask the bitmap to a smaller width (e.g. 1.2.3-era 32-bit, 1.5/1.9-era 41-bit, GLoarbLine 55-bit) will not see the bit; servers MUST treat its absence on such accounts as "not granted" and MUST NOT silently widen legacy accounts. Operators upgrading to this extension grant the bit explicitly per account or per template.

For private chat rooms, the server MUST verify that the sender is a member of the room before accepting a chat transaction that references a media handle.

For `TranDownloadMedia`, the server MUST track the set of authorized recipients for each handle. The set is fixed at the moment the chat transaction is relayed:

- Public chat handle → all clients that were connected and had read access at relay time.
- Private message handle → the original sender and the single recipient.
- Private chat room handle → the sender and the room's membership at relay time.

Subsequently joining a room or gaining read access does NOT grant retroactive download rights. Servers MAY narrow this set further (e.g. by removing departed users) but MUST NOT widen it.

### Relaying to Capable Clients

When relaying a chat transaction with a media handle to a recipient that has negotiated `CAPABILITY_INLINE_MEDIA`:

1. Include `DATA_CHAT_MEDIA_ID`, the canonical `DATA_CHAT_MEDIA_TYPE`, and the canonical `DATA_CHAT_MEDIA_WIDTH`, `DATA_CHAT_MEDIA_HEIGHT`, `DATA_CHAT_MEDIA_BYTES` server-supplied fields.
2. Format and include `DATA_DATA` as for any other chat message.
3. Add the recipient to the handle's authorization set.

### Legacy Client Fallback

When relaying to a recipient that has not negotiated the capability:

1. Strip all media fields.
2. Relay the text field as-is. Do NOT substitute a gateway URL except in the public-chat case described below.
3. Do NOT add the recipient to the handle's authorization set.

### Media Gateway (optional, public contexts only)

The media gateway is an optional, operator-controlled feature. Its purpose is to give legacy clients in **public chat** a clickable URL instead of a `[image]` placeholder. It is not part of the protocol; it is a server-side configuration detail.

The following constraints MUST hold for any conformant gateway integration:

- **Public contexts only.** Gateway upload MUST NOT occur for private messages or private chat rooms under any circumstance, even if all recipients in such a context are legacy. The fallback in private contexts is the text field as the sender supplied it.
- **Only when needed.** Gateway upload MUST NOT occur if every recipient negotiated the capability. The server MUST upload at most once per chat transaction, regardless of the number of legacy recipients.
- **Canonical bytes only.** The bytes uploaded to the gateway MUST be the canonical bytes produced by the validation pipeline, never the original uploaded bytes.
- **Unguessable URLs.** The gateway URL MUST contain at least 128 bits of entropy in its path component. Servers SHOULD prefer gateways that set `X-Robots-Tag: noindex`.
- **Operator notice.** Operators enabling the gateway MUST be informed (via configuration documentation) that gateway uploads make the canonical image publicly retrievable for the gateway's retention period.
- **Off by default.** Gateway integration MUST default to disabled.

When the gateway substitution applies, the server replaces the text fallback in `DATA_DATA` (typically `[image]`) with the gateway URL before relaying to legacy recipients only. Capable recipients still receive the original text and the media handle.

### Resource Limits

Servers MUST enforce, and SHOULD expose as configuration:

| Limit | Recommended Default | Notes |
|---|---|---|
| Maximum encoded payload | 256 KB | After all chunks are assembled; before the validation pipeline runs. |
| Maximum dimensions | 2048 × 2048 | Reject before full decode. |
| Maximum decoded pixel count | ~4.2 megapixels | Belt-and-braces with the dimensions check. |
| Maximum animation frames | 150 | For animated formats. |
| Maximum animation duration | 15 seconds | For animated formats. |
| Per-account upload rate | 1 image per 10 seconds | Reject excess with a generic rate-limit error. |
| Per-account hourly volume | 30 images / hour | |
| Per-IP hourly volume | 100 images / hour | |
| Handle lifetime | 24 hours | Handles MUST be discarded after this window. |
| Decode time budget | 2 seconds | Hard cap; reject on timeout. |
| Decode memory budget | 64 MB | Reject on exhaustion. |
| Upload-session idle timeout | 30 seconds | Discard partial chunks if the next chunk does not arrive in time. |

Servers MAY tighten these defaults. Servers SHOULD NOT relax them without operator awareness.

### Logging and Privacy

- Servers MUST NOT log the raw or canonical image bytes to general operational logs by default. Operators MAY enable byte-level logging only via an explicit, documented configuration flag.
- Servers MAY log a metadata-only placeholder (e.g. `[image: <mime>, <bytes> bytes, <width>x<height>]`) for chat-history and audit purposes.
- Servers MUST NOT include media bytes or media handles in any unauthenticated channel (status pages, public APIs, etc.).
- Servers SHOULD redact media handles in logs that are shared outside the operator's trust boundary.

### Storage and Lifetime

- Stored canonical bytes MUST be removed when the handle expires (see [Resource Limits](#resource-limits)).
- Servers MAY delete handles earlier on operator demand (e.g. moderation tooling).
- Servers SHOULD scrub the underlying storage so that deleted bytes cannot be recovered from disk after expiry.

## Client Behaviour

### Sending Media

1. Confirm `CAPABILITY_INLINE_MEDIA` is active for the session. If not, suppress the media UI.
2. Read the source image and SHOULD strip metadata locally (in particular EXIF GPS) as a defence-in-depth measure. The server will strip metadata regardless.
3. If the encoded size exceeds the server's advertised maximum (or the client's own conservative default), the client SHOULD resize or recompress before uploading.
4. Send `TranUploadMedia` with the bytes. If the bytes do not fit in a single field, use the chunked-upload convention.
5. On success, take the returned `DATA_CHAT_MEDIA_ID` and `DATA_CHAT_MEDIA_TYPE` and include them in the chat transaction (`TranChatSend` / `TranSendInstantMsg`) along with the user's text.
6. On error, surface a generic message to the user. Do not retry automatically more than once.

Clients MUST NOT cache media handles across sessions. Handles are session-scoped from the client's perspective even if the server permits longer lifetimes.

### Receiving Media

1. On `TranChatMsg` / `TranServerMsg` containing both `DATA_CHAT_MEDIA_ID` and `DATA_CHAT_MEDIA_TYPE`, render a placeholder using the server-supplied dimensions and byte size, then issue `TranDownloadMedia` to fetch the bytes.
2. Decode the returned bytes using a memory-safe decoder configured with hard limits matching or stricter than the server's. Clients MUST NOT trust server-supplied dimensions as a substitute for client-side limits.
3. Render the image inline at a reasonable display size. Do not stretch beyond intrinsic resolution.
4. If decoding fails, display the text fallback only and SHOULD warn the user that the image could not be displayed.
5. Clients MUST treat the canonical MIME type, not the file extension or any embedded metadata, as authoritative for choosing a decoder.

### Supported Formats

Clients implementing this extension MUST support decoding:

| Format | Canonical MIME | Notes |
|---|---|---|
| JPEG | `image/jpeg` | Baseline and progressive. |
| PNG | `image/png` | Including alpha. |
| GIF | `image/gif` | Static and animated. |

Clients MAY support additional formats, but MUST NOT render any format the negotiated server is not known to canonicalise. Servers MUST NOT canonicalise to formats outside the set above without a corresponding capability extension.

The following formats MUST NOT be accepted by the server and MUST NOT be rendered by the client under this capability bit:

- SVG (XML-based, scriptable, network-fetching).
- WebP, AVIF, HEIC (decoder hardening varies widely; defer to a future capability bit).
- Any container-style image format (TIFF, ICO with embedded payloads).

## Security Considerations

This section enumerates the threats this extension is designed to mitigate and the residual risks operators should be aware of.

### Threats mitigated by the design

- **Embedding attack content in chat fields.** Bytes are never relayed inside a chat transaction; only canonical bytes from a server-controlled re-encode pipeline are served, and only to authorized recipients.
- **MIME confusion / polyglot files.** The server sniffs magic bytes, rejects trailing data after the format's natural EOF, and re-encodes; the canonical bytes cannot be a valid container of a second format.
- **Decoder exploits in clients.** The server's re-encode replaces unusual but format-legal byte patterns (oversized markers, malformed chunks, hostile ICC profiles) with output from a known-good encoder.
- **Decompression bombs.** Dimension probing precedes full decode; full decode runs under explicit memory and time budgets.
- **Metadata leakage.** The re-encode discards EXIF, ICC, XMP, PNG text chunks, and GIF comments unconditionally.
- **Unauthorized retrieval.** Handles are unguessable, and the server enforces a fixed authorization set captured at relay time.
- **DM leakage to public hosts.** The media gateway is restricted to public-chat fallback only and is off by default.
- **Amplification flooding.** Per-account and per-IP rate and volume limits, plus a fixed handle lifetime, bound the cost of a hostile uploader.
- **Capability spoofing.** Servers strip media fields from senders and recipients that did not negotiate the capability.

### Residual risks operators should weigh

- **Storage cost.** Each handle holds canonical bytes for up to the handle lifetime. Operators with large user populations should size storage and may tighten the lifetime or per-account volume.
- **Re-encode CPU cost.** The validation pipeline is the dominant per-message cost. Operators on constrained hardware should tighten the encoded-size and dimension caps.
- **Plaintext transport.** Hotline sessions may run without transport encryption. Operators MUST be informed that, in the absence of HOPE or TLS, image bytes traverse the wire in the clear between server and clients. This extension does not change that property; it only constrains what the server itself does with the bytes.
- **Gateway operator trust.** When the gateway is enabled, the gateway operator can read all public-chat images and retains them for their own retention period. Operators who do not control the gateway are extending trust to a third party.
- **Client decoder weaknesses.** The server's re-encode reduces but does not eliminate the risk of a client-side decoder bug. Clients SHOULD prefer memory-safe decoders and SHOULD apply their own limits.

### Required server behaviour summary

For convenience, the following are normative MUSTs concentrated in one place:

- MUST validate, canonicalise, and re-encode every uploaded image before relaying.
- MUST strip all metadata from canonical bytes.
- MUST issue handles with at least 128 bits of entropy.
- MUST authorize every download against a fixed set captured at relay time.
- MUST NOT relay original bytes.
- MUST NOT distinguish "expired" from "unauthorized" in error responses.
- MUST NOT use the media gateway for private messages or private chat rooms.
- MUST default the gateway and `AccessSendMedia` to off.
- MUST drop media fields from senders or recipients that did not negotiate the capability.

## Implementation Notes

- **No new chat transactions.** This extension introduces only the upload and download transactions. Public chat (105/106), private messages (108/104), and private chat rooms (105/106 with `DATA_CHATID`) all continue to use their existing types, with two added companion fields and three server-supplied advisory fields.
- **Field-size discipline.** All fields conform to standard Hotline 16-bit length encoding. Image bytes are exchanged via dedicated transactions that support multi-chunk replies; no field ever exceeds 65,535 bytes.
- **Capability echo on relay.** The server is the single place where capability checks are applied to outbound chat fan-out. Clients MUST NOT attempt to detect peer capabilities themselves.
- **Chat logging.** Chat history (see the chat-history extension) MUST treat media as a metadata-only entry by default; canonical bytes MAY be retained separately under operator policy.
- **Mixed audiences in private rooms.** When a private chat room contains capable and legacy members, the server constructs two relay variants: capable members receive the media fields; legacy members receive the text only. Gateway substitution is NOT used in this case.
- **Capability bit.** This extension uses bit 3 (`0x0008`) of `DATA_CAPABILITIES`. Servers that do not implement this extension MUST NOT confirm the bit, even if a client advertises it.

---

## Capability Bit

| Bit | Mask | Name | Description |
|---|---|---|---|
| 3 | `0x0008` | `CAPABILITY_INLINE_MEDIA` | Client supports inline image attachments in chat via the upload/download transactions described in this document. |

---

Status: draft; subject to refinement.
