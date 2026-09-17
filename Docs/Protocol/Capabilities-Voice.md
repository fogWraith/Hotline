# Voice Chat Extension

> Last updated: September 17, 2026

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document describes the voice chat extension to the Hotline protocol. It adds real-time voice communication using a server-side SFU (Selective Forwarding Unit) architecture, where all media routes through the server. Clients negotiate voice support during login, establish WebRTC peer connections to the server, and the server forwards audio streams between participants in the same chat room.

Media is carried over DTLS-SRTP by default. For clients on platforms with no TLS stack — the classic Mac OS clients the protocol comes from — an operator may additionally allow the [plain RTP transport](#plain-rtp-transport), which keeps the signaling, the SDP and the ICE binding but sends audio in the clear.

For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md).

## Table of Contents

- [Background](#background)
- [Architecture](#architecture)
  - [Why SFU](#why-sfu)
  - [Why Not Peer-to-Peer](#why-not-peer-to-peer)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
  - [Capability Bit](#capability-bit)
  - [Server Configuration](#server-configuration)
- [Codec Negotiation](#codec-negotiation)
  - [Supported Codecs](#supported-codecs)
  - [Codec ID Table](#codec-id-table)
  - [Codec Selection](#codec-selection)
  - [Codec Profiles](#codec-profiles)
- [Signaling](#signaling)
  - [Overview](#overview)
  - [Transaction Semantics](#transaction-semantics)
  - [Transaction Types](#transaction-types)
  - [Data Objects](#data-objects)
  - [SDP Format](#sdp-format)
  - [ICE Candidate Format](#ice-candidate-format)
  - [Join Voice Room (600)](#join-voice-room-600)
  - [Leave Voice Room (601)](#leave-voice-room-601)
  - [Voice SDP Offer (602)](#voice-sdp-offer-602)
  - [Voice SDP Answer (603)](#voice-sdp-answer-603)
  - [Voice ICE Candidate (604)](#voice-ice-candidate-604)
  - [End of ICE Candidates](#end-of-ice-candidates)
  - [Voice Room Status (605)](#voice-room-status-605)
  - [Voice Mute (606)](#voice-mute-606)
  - [Hand-Rolled Signaling](#hand-rolled-signaling)
- [Media Transport](#media-transport)
  - [UDP Port](#udp-port)
  - [WebRTC Session](#webrtc-session)
  - [Track-to-User Mapping](#track-to-user-mapping)
  - [Renegotiation Flow](#renegotiation-flow)
  - [Session Timeout and Failure](#session-timeout-and-failure)
  - [Stream Topology](#stream-topology)
  - [DTLS and SRTP](#dtls-and-srtp)
  - [Plain RTP Transport](#plain-rtp-transport)
  - [RTP and RTCP](#rtp-and-rtcp)
- [Room Model](#room-model)
- [Access Privileges](#access-privileges)
  - [Room Membership](#room-membership)
- [Bandwidth Considerations](#bandwidth-considerations)
- [Client Behaviour](#client-behaviour)
- [Server Behaviour](#server-behaviour)
- [Implementation Notes](#implementation-notes)

---

## Background

The original Hotline protocol supports text chat in public and private chat rooms but has no mechanism for voice communication. As voice-over-IP became ubiquitous, adding voice to Hotline's existing chat room model is a natural fit — users already gather in rooms with join/leave semantics, making voice an incremental enhancement rather than a separate system.

This extension uses WebRTC for media transport, leveraging its built-in encryption (DTLS/SRTP), codec support, and NAT traversal. By using the server as an SFU, the architecture avoids the need for external TURN servers — clients already know the server's address and can establish direct UDP connections to it.

---

## Architecture

### Why SFU

In a Selective Forwarding Unit model, each client sends a single audio stream to the server, and the server forwards copies of that stream to every other participant in the room. The server performs no mixing or transcoding — it simply routes packets.

```
        ┌─────────────────────────┐
        │         Server          │
        │                         │
        │   ┌─────────────────┐   │
        │   │   SFU / Router  │   │
        │   └──┬────┬────┬────┘   │
        │      │    │    │        │
        └──────┼────┼────┼────────┘
               │    │    │
          ┌────┘    │    └────┐
          │         │         │
       Client A  Client B  Client C
```

Each client sends 1 stream and receives N−1 streams, where N is the number of active speakers in the room.

Advantages:
- **Low server CPU** — no mixing or transcoding; just packet forwarding
- **Low latency** — no processing delay from mixing
- **Individual stream control** — clients can independently adjust volume per speaker, or mute specific participants
- **Scales well** — adding a participant adds one inbound and N outbound stream copies, which is linear

### Why Not Peer-to-Peer

A peer-to-peer mesh (each client connects directly to every other client) avoids the server entirely but introduces significant problems:

- **NAT traversal** — clients behind NATs cannot reach each other without STUN/TURN infrastructure
- **Upload bandwidth** — each client must send N−1 copies of their stream, scaling poorly
- **Firewall complexity** — operators must configure additional infrastructure for STUN/TURN

Since every Hotline client already maintains a TCP connection to the server, the SFU model requires no additional infrastructure — only a UDP port on the same host.

---

## Compatibility and Negotiation

### Capability Bit

| Bit | Mask | Name | Description |
|---|---|---|---|
| 2 | `0x0004` | `CAPABILITY_VOICE` | Client supports voice chat via WebRTC SFU |

This bit is defined in the `DATA_CAPABILITIES` bitmask (field `0x01F0`). See [DATA_CAPABILITIES](Capabilities.md) for the general negotiation flow.

When all three extensions are active (large files, text encoding, voice), the capability bitmask is `0x0007`.

**Negotiation:**

1. Client sets bit 2 in `DATA_CAPABILITIES` during Login (107).
2. Server checks its configuration. If voice is enabled and the server has a functioning SFU, it echoes bit 2 back in the login reply.
3. If the server does not echo bit 2, voice is unavailable for the session. The client MUST NOT display voice UI elements or send voice-related transactions.

Clients that do not set bit 2 are unaffected — they participate in text chat normally and never receive voice-related transactions.

### Server Configuration

The server operator controls voice via configuration, using Janus as an example:

| Setting | Type | Default | Description |
|---|---|---|---|
| `EnableVoice` | bool | `false` | Master switch for the voice subsystem |
| `VoiceUDPPort` | int | `0` | UDP port for WebRTC media. `0` = base port + 4 |
| `VoiceMaxPerRoom` | int | `16` | Maximum simultaneous voice participants per room |
| `VoiceAllowPlainRTP` | bool | `false` | Allow clients that ask for it to use the [plain RTP transport](#plain-rtp-transport) |


---

## Codec Negotiation

### Supported Codecs

| Codec | Clock Rate | Channels | Typical Bitrate | Use Case |
|---|---|---|---|---|
| **G.711 μ-law (PCMU)** | 8000 Hz | 1 (mono) | 64 kbps | Universal codec, zero-configuration, no licensing, minimal CPU |

G.711 μ-law (PCMU) is the only supported codec. It is a mandatory-to-implement codec in every WebRTC stack (RFC 7874), requires no dynamic payload type negotiation, and adds zero encode/decode complexity. The fixed 64 kbps bitrate is acceptable for the small room sizes typical of Hotline voice chat.

### Codec ID Table

Codec IDs are used in the `DATA_VOICE_PARTICIPANTS` binary structure and in server-internal state. This table is the canonical reference:

| Codec ID | Name | SDP Encoding Name | Clock Rate | Channels | RTP Payload Type |
|---|---|---|---|---|---|
| 0 | PCMU | `PCMU` | 8000 | 1 | 0 (static, RFC 3551) |
| 1–65535 | Reserved | — | — | — | — |

PCMU uses static RTP payload type 0 as defined in RFC 3551. No dynamic payload type negotiation is required.

### Codec Selection

Codec selection follows WebRTC's standard SDP offer/answer model:

1. When a client joins a voice room, the server sends an SDP offer listing PCMU as the codec.
2. The client's SDP answer confirms PCMU support.
3. WebRTC's built-in negotiation activates the codec.

Since PCMU is the only supported codec, all participants in a room always use the same codec and no transcoding is ever required.

### Codec Profiles

PCMU has a fixed profile with no configuration required:

| Parameter | Value | Notes |
|---|---|---|
| Bitrate | 64000 bps | Fixed; 8000 samples/sec × 8 bits/sample |
| Frame size | 20 ms | 160 samples per frame, standard for voice |
| Channels | 1 (mono) | Voice does not benefit from stereo |
| Sample rate | 8000 Hz | Standard telephony rate |
| Encoding | μ-law companding | ITU-T G.711, 8-bit logarithmic quantisation |

#### G.711 μ-law Encoding Reference

G.711 μ-law (PCMU) compresses 16-bit linear PCM samples to 8-bit logarithmic values using the μ-law companding curve defined in ITU-T Recommendation G.711 and profiled for RTP in [RFC 3551 §4.5.14](https://datatracker.ietf.org/doc/html/rfc3551#section-4.5.14).

**Encoding (16-bit signed linear → 8-bit μ-law):**

```
μ-law compression formula (μ = 255):

  F(x) = sgn(x) · ln(1 + μ|x|) / ln(1 + μ)

where x is the normalised input sample [-1.0, 1.0].
```

In practice, implementations use a segment-based lookup table rather than floating-point logarithms. The algorithm:

1. Clamp the 16-bit signed input to the range [−32635, +32635].
2. Add a bias of 0x84 (132) to the absolute value.
3. Determine the segment (exponent) from the position of the most significant bit.
4. Extract 4 mantissa bits from below the MSB.
5. Combine sign (1 bit), exponent (3 bits), and mantissa (4 bits) into an 8-bit value.
6. Invert all bits (one's complement) for transmission.

**Decoding (8-bit μ-law → 16-bit signed linear):**

1. Invert all bits of the received byte.
2. Extract sign, exponent, and mantissa.
3. Reconstruct the linear value: `((mantissa << 1) + 33) << exponent) - 33`.
4. Apply the sign bit.

A complete C reference implementation is available in ITU-T G.711 Annex A and in [RFC 3551 Appendix](https://datatracker.ietf.org/doc/html/rfc3551). Most WebRTC libraries (including Pion, libwebrtc, and GStreamer) include built-in G.711 codecs and handle encoding/decoding transparently.

Implementors writing custom audio pipelines (e.g. capturing raw PCM from a microphone API) MUST encode to μ-law before packetising RTP, and MUST decode received μ-law samples to linear PCM before playback.

---

## Signaling

### Overview

Voice signaling uses the existing Hotline TCP connection. No additional signaling channel is required. Six new transaction types handle room join/leave, WebRTC SDP exchange, ICE candidate exchange, room status updates, and mute state.

All signaling transactions are only sent to/from clients that have negotiated `CAPABILITY_VOICE`.

### Transaction Semantics

Voice transactions use the standard Hotline transaction framing (see [Hotline.md](Hotline.md)). A few semantics require clarification for implementors:

- **Request/reply transactions** (600, 601, 603, 606): The client sends a transaction with a unique task ID and the "is reply" flag unset. The server responds with a transaction using the same task ID and the "is reply" flag set.
- **Server-initiated notifications** (602, 604, 605): The server sends these asynchronously with task ID `0` and the "is reply" flag **unset**. The client does not send a reply. These arrive interleaved with normal transaction traffic on the TCP connection.
- **Client-initiated notifications** (604): When the client sends an ICE candidate, it uses task ID `0` and does not expect a reply.

Implementors should dispatch incoming transactions by type ID. Server-initiated notifications (task ID 0, not a reply) should be handled as events, not matched against a pending request queue.

### Transaction Types

| ID | Name | Direction | Description |
|---|---|---|---|
| 600 | Join Voice Room | Client → Server | Request to join voice in a chat room |
| 601 | Leave Voice Room | Client → Server | Leave voice in a chat room |
| 602 | Voice SDP Offer | Server → Client | Server's SDP offer for WebRTC negotiation |
| 603 | Voice SDP Answer | Client → Server | Client's SDP answer |
| 604 | Voice ICE Candidate | Bidirectional | ICE candidate exchange |
| 605 | Voice Room Status | Server → Client | Notification of voice participants and state changes |
| 606 | Voice Mute | Client → Server | Toggle mute state |

Transaction IDs 600–606 are chosen to avoid collision with existing Hotline transaction types (which range from 101–355 in the base protocol).

### Data Objects

| ID (hex) | Name | Type | Description |
|---|---|---|---|
| `0x01F5` | `DATA_VOICE_SDP` | String | SDP blob — see [SDP Format](#sdp-format) |
| `0x01F6` | `DATA_VOICE_ICE` | String | JSON-encoded ICE candidate — see [ICE Candidate Format](#ice-candidate-format) |
| `0x01F7` | `DATA_VOICE_CODEC` | String | Active codec name for the room |
| `0x01F8` | `DATA_VOICE_MUTED` | UInt16 | Mute state: 0 = unmuted, 1 = muted |
| `0x01F9` | `DATA_VOICE_PARTICIPANTS` | Binary | Packed array of voice participant entries |
| `0x01FB` | `DATA_VOICE_TRANSPORT` | UInt16 | Media transport selector — see [Plain RTP Transport](#plain-rtp-transport). `0` = DTLS-SRTP (default), `1` = plain RTP. `0x01FA` is the large-file extension's resume digest. |

`DATA_VOICE_PARTICIPANTS` is a packed binary structure:

```
For each participant (6 bytes):
  [2] User ID    (big-endian uint16)
  [2] Flags      (big-endian uint16: bit 0 = muted, bits 1-15 reserved)
  [2] Codec ID   (big-endian uint16: see Codec ID Table)
```

The total field length divided by 6 gives the participant count. See [Codec ID Table](#codec-id-table) for codec ID values.

### SDP Format

`DATA_VOICE_SDP` contains a standard WebRTC Session Description Protocol blob as defined in RFC 8866 (SDP) and the WebRTC JSEP specification (RFC 8829). The value is a **UTF-8 encoded string** using standard SDP line-based formatting (`v=`, `o=`, `s=`, `m=`, `a=`, etc.).

Implementors should use their platform's WebRTC library to generate and parse SDP. The SDP is not a custom format — any RFC-compliant WebRTC stack will produce compatible output.

The server's SDP offer MUST contain:
- One `m=audio` section per receive track (one per existing voice participant in the room)
- One `m=audio` section for the joiner's send track
- `a=mid` attributes labelling each media section (see [Track-to-User Mapping](#track-to-user-mapping))
- `a=rtpmap:0 PCMU/8000` — the only codec offered
- `a=fingerprint` for DTLS key verification
- `a=ice-lite` at session level — the server is an ICE-lite agent (RFC 8445 §2.5): it offers only host candidates and never initiates connectivity checks
- `a=ice-ufrag` and `a=ice-pwd` for ICE authentication
- `a=ssrc:<ssrc> cname:<cname>` on every `user-{UID}` section, declaring the SSRC the client will observe on RTP packets carrying that user's audio (see [Track-to-User Mapping](#track-to-user-mapping))
- `a=group:BUNDLE` — all media sections MUST be bundled over a single transport (RFC 8843)
- `a=rtcp-mux` — RTCP MUST be multiplexed on the same port as RTP (RFC 5761)
- `a=setup:actpass` on the offer, `a=setup:active` on the answer — DTLS role negotiation (RFC 8842)
- A direction attribute on each media section: `a=sendonly` or `a=recvonly`, or `a=inactive` on the section of a participant who has left (see [Track-to-User Mapping](#track-to-user-mapping))

The `m=audio` line uses port `9`, which is the standard placeholder port in bundled WebRTC SDP (RFC 8843 §9.3). Port `9` has no transport-layer significance — actual media transport uses the ICE candidate addresses. Implementations MUST NOT attempt to connect to port 9.

The server's offer MAY carry further attributes that a WebRTC stack emits by habit — `a=rtcp-fb`, `a=extmap`, `a=extmap-allow-mixed`, `a=rtcp-rsize`, `a=msid`, `a=rtpmap:0 PCMU/8000/1` with an explicit channel count — none of which this specification depends on. A client MUST ignore any it does not implement and MAY omit them from its answer.

#### Annotated SDP Offer Example

The following is a complete SDP offer the server would send to User 5 joining a room where users 12 and 23 are already in voice. Annotations (lines starting with `#`) are not part of the SDP.

Note that direction attributes are always written from the perspective of the party that authored the description. In the server's offer, sections carrying other users' audio are `a=sendonly` (the server sends) and the client's microphone section is `a=recvonly` (the server receives); the client's answer mirrors each direction.

```
# Session-level attributes
v=0
o=- 1234567890 1 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-lite
a=group:BUNDLE user-12 user-23 send
a=msid-semantic: WMS

# Audio from user 12 (server sends, client receives)
m=audio 9 UDP/TLS/RTP/SAVPF 0
c=IN IP4 0.0.0.0
a=mid:user-12
a=rtpmap:0 PCMU/8000
a=sendonly
a=rtcp-mux
a=setup:actpass
a=ice-ufrag:srvr
a=ice-pwd:servericepasswordvalue1234
a=fingerprint:sha-256 AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99
a=ssrc:1862706922 cname:voice-12
a=candidate:1 1 udp 2130706431 198.51.100.7 5504 typ host
a=end-of-candidates

# Audio from user 23 (server sends, client receives)
m=audio 9 UDP/TLS/RTP/SAVPF 0
c=IN IP4 0.0.0.0
a=mid:user-23
a=rtpmap:0 PCMU/8000
a=sendonly
a=rtcp-mux
a=setup:actpass
a=ice-ufrag:srvr
a=ice-pwd:servericepasswordvalue1234
a=fingerprint:sha-256 AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99
a=ssrc:3467440883 cname:voice-23

# Microphone section for the joining client (client sends, server receives)
m=audio 9 UDP/TLS/RTP/SAVPF 0
c=IN IP4 0.0.0.0
a=mid:send
a=rtpmap:0 PCMU/8000
a=recvonly
a=rtcp-mux
a=setup:actpass
a=ice-ufrag:srvr
a=ice-pwd:servericepasswordvalue1234
a=fingerprint:sha-256 AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99
```

#### Annotated SDP Answer Example

The client's answer mirrors the offer structure, confirming each media section:

```
v=0
o=- 9876543210 1 IN IP4 0.0.0.0
s=-
t=0 0
a=group:BUNDLE user-12 user-23 send
a=msid-semantic: WMS

# Accept receive track for user 12
m=audio 9 UDP/TLS/RTP/SAVPF 0
c=IN IP4 0.0.0.0
a=mid:user-12
a=rtpmap:0 PCMU/8000
a=recvonly
a=rtcp-mux
a=setup:active
a=ice-ufrag:clnt
a=ice-pwd:clienticepasswordvalue5678
a=fingerprint:sha-256 11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00

# Accept receive track for user 23
m=audio 9 UDP/TLS/RTP/SAVPF 0
c=IN IP4 0.0.0.0
a=mid:user-23
a=rtpmap:0 PCMU/8000
a=recvonly
a=rtcp-mux
a=setup:active
a=ice-ufrag:clnt
a=ice-pwd:clienticepasswordvalue5678
a=fingerprint:sha-256 11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00

# Accept send track — the a=ssrc line declaring the microphone stream is REQUIRED
# (see Send SSRC Declaration below)
m=audio 9 UDP/TLS/RTP/SAVPF 0
c=IN IP4 0.0.0.0
a=mid:send
a=rtpmap:0 PCMU/8000
a=sendonly
a=rtcp-mux
a=setup:active
a=ice-ufrag:clnt
a=ice-pwd:clienticepasswordvalue5678
a=fingerprint:sha-256 11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00
a=ssrc:2226456186 cname:janusclientmic01
```

Key differences from the offer: `a=setup:active` (the answerer takes the DTLS client role), the client's own ICE credentials and DTLS fingerprint replace the server's, and the send section declares the SSRC of the client's microphone stream. The `a=ssrc` lines on the `user-{UID}` sections of the offer are not mirrored — they describe what the server sends.

#### Send SSRC Declaration

The client's answer MUST declare the SSRC of its microphone stream in the `send` media section with an `a=ssrc` attribute ([RFC 5576](https://datatracker.ietf.org/doc/html/rfc5576)):

```
a=ssrc:<ssrc> cname:<cname>
```

`<ssrc>` is the 32-bit synchronisation source identifier the client will use in its outbound RTP packets, as an unsigned decimal integer. It MUST match the SSRC field of the RTP packets the client actually sends. `<cname>` is the RTCP canonical name (RFC 3550 §6.5.1); any stable, randomly generated string is acceptable. Standard WebRTC stacks (libwebrtc, browsers, pion) emit this attribute automatically when a sending track is attached; implementations that hand-write SDP must take care not to omit it.

Rationale: the server is always the offerer, and this specification requires no RTP header extensions, so the `a=ssrc` declaration in the answer is the only mechanism by which the server can attribute inbound RTP packets to the client's microphone track. A conforming server tolerates answers that omit the declaration by falling back to payload-type matching, but that fallback causes an additional unlabelled media section to appear in the server's subsequent renegotiation offers (which clients must ignore — see [Track-to-User Mapping](#track-to-user-mapping)). Clients MUST NOT rely on the fallback.

#### SDP Size Considerations

SDP blobs are carried in `DATA_VOICE_SDP` (field `0x01F5`) as a standard Hotline data field. The maximum field size is 65535 bytes (uint16 length prefix in the Hotline framing). In practice, a voice-only SDP for 16 participants is well under 10 KB. Implementations SHOULD NOT generate SDP larger than 32 KB to allow headroom for future extensions.

### ICE Candidate Format

`DATA_VOICE_ICE` contains a JSON object with the following fields, matching the standard `RTCIceCandidateInit` dictionary from the WebRTC API:

```json
{
  "candidate": "candidate:1 1 UDP 2130706431 192.0.2.1 5004 typ host",
  "sdpMid": "0",
  "sdpMLineIndex": 0,
  "usernameFragment": "abc123"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `candidate` | string | Yes | The SDP candidate-attribute string (RFC 8839). Empty string signals end-of-candidates. |
| `sdpMid` | string | Yes | The media stream identification tag from the SDP, matching an `a=mid` value. |
| `sdpMLineIndex` | integer | No | Zero-based index of the `m=` line in the SDP. Fallback if `sdpMid` is not available. |
| `usernameFragment` | string | No | ICE username fragment for candidate association. |

This format is directly compatible with the `RTCIceCandidateInit` type in browser WebRTC APIs, `pion/webrtc` in Go, `libwebrtc` in C++/Swift, and equivalent libraries on other platforms.

### Join Voice Room (600)

**Client → Server**

The client requests to join voice chat in a specific room.

| Field | ID | Type | Required | Notes |
|---|---|---|---|---|
| Chat ID | 114 | UInt32 | Yes | Chat room ID. `0` = public chat. |
| Voice Transport | `0x01FB` | UInt16 | No | `0` = DTLS-SRTP (the default when absent), `1` = [plain RTP](#plain-rtp-transport). |

**Server Reply (success):**

| Field | ID | Type | Notes |
|---|---|---|---|
| Chat ID | 114 | UInt32 | Echoed back |
| Voice SDP | `0x01F5` | String | Server's SDP offer |
| Voice Codec | `0x01F7` | String | Room's active codec (e.g. `"PCMU"`) |
| Voice Participants | `0x01F9` | Binary | Current voice participants |
| Voice Transport | `0x01FB` | UInt16 | The transport in use for this session. Servers that predate this field omit it, which means `0`. |

**Server Reply (error):**

Standard error reply with `DATA_ERROR_TEXT` (field 100). Error strings are human-readable and not intended for programmatic parsing. Clients SHOULD display the error text to the user as-is. Common error conditions include:
- Room is full (`VoiceMaxPerRoom` exceeded)
- Voice is disabled on the server
- Insufficient privileges (`accessVoiceChat` not set)
- Plain RTP requested but not allowed (`VoiceAllowPlainRTP` is off), or an unknown transport value

A transport the server will not provide is refused **before** any room state changes: the client is neither added to the requested room nor removed from one it is already in. The server MUST NOT substitute a different transport for the one requested — a client that asks for plain RTP has no DTLS stack to fall back to.

The server also sends a **Voice Room Status (605)** notification to all other voice participants in the room to announce the new joiner.

### Leave Voice Room (601)

**Client → Server**

| Field | ID | Type | Required | Notes |
|---|---|---|---|---|
| Chat ID | 114 | UInt32 | Yes | Room to leave voice in |

**Server Reply:** Empty success reply (no fields).

The server tears down the WebRTC peer connection for this client in the specified room and sends a **Voice Room Status (605)** notification to remaining participants.

If a client disconnects without sending Leave Voice Room, the server MUST clean up automatically.

### Voice SDP Offer (602)

**Server → Client (notification)**

Sent by the server when it needs to initiate or renegotiate the WebRTC session. This occurs:
- In the Join Voice Room reply (initial offer)
- When the room's participant list changes and renegotiation is needed (a user joins, leaves, or rejoins voice)
- Immediately after a Voice SDP Answer (603), if participant changes accumulated while that offer was outstanding (see [Renegotiation Flow](#renegotiation-flow))

| Field | ID | Type | Notes |
|---|---|---|---|
| Chat ID | 114 | UInt32 | Room context |
| Voice SDP | `0x01F5` | String | SDP offer blob |

### Voice SDP Answer (603)

**Client → Server**

The client's response to an SDP offer.

| Field | ID | Type | Required | Notes |
|---|---|---|---|---|
| Chat ID | 114 | UInt32 | Yes | Room context |
| Voice SDP | `0x01F5` | String | SDP answer blob |

**Server Reply:** Empty success reply. Media flow begins once the WebRTC session is established.

### Voice ICE Candidate (604)

**Bidirectional (notification)**

ICE candidates are exchanged as trickle-ICE after the SDP offer/answer. Both client and server send these as notifications (no reply expected).

| Field | ID | Type | Notes |
|---|---|---|---|
| Chat ID | 114 | UInt32 | Room context |
| Voice ICE | `0x01F6` | String | JSON-encoded ICE candidate (see [ICE Candidate Format](#ice-candidate-format)) |

Since the client connects directly to the server (whose address is already known), ICE negotiation is typically trivial — the server's host candidate is the only one needed. However, the full ICE exchange is preserved for correctness in edge cases (e.g. server behind a load balancer).

### End of ICE Candidates

After sending all ICE candidates, each side signals completion by sending a Voice ICE Candidate (604) with an **empty string** in `DATA_VOICE_ICE`:

| Field | ID | Type | Value |
|---|---|---|---|
| Chat ID | 114 | UInt32 | Room context |
| Voice ICE | `0x01F6` | String | `""` (empty — zero-length string) |

The end-of-candidates signal as a complete JSON object:

```json
{
  "candidate": "",
  "sdpMid": "send",
  "sdpMLineIndex": 0
}
```

The `sdpMid` and `sdpMLineIndex` fields SHOULD reference the first media section, but their values are not semantically significant for end-of-candidates — the empty `candidate` string is the authoritative signal.

Alternatively, an empty string `""` (zero-length `DATA_VOICE_ICE` field) is also accepted as an end-of-candidates signal.

This follows the standard WebRTC end-of-candidates convention. Implementations MUST NOT wait for this signal before attempting connectivity checks — trickle ICE allows checks to begin as candidates arrive.

### Voice Room Status (605)

**Server → Client (notification)**

Sent to all voice participants in a room when the participant list changes (join, leave, mute/unmute).

| Field | ID | Type | Notes |
|---|---|---|---|
| Chat ID | 114 | UInt32 | Room context |
| Voice Participants | `0x01F9` | Binary | Updated participant list |

### Voice Mute (606)

**Client → Server**

Toggles the client's mute state. When muted, the server stops forwarding the client's audio stream to other participants.

| Field | ID | Type | Required | Notes |
|---|---|---|---|---|
| Chat ID | 114 | UInt32 | Yes | Room context |
| Voice Muted | `0x01F8` | UInt16 | Yes | 0 = unmuted, 1 = muted |

**Server Reply:** Empty success reply.

The server sends a **Voice Room Status (605)** notification to all participants reflecting the updated mute state.

**Server-side mute enforcement:** When a client is muted, the server MUST discard incoming audio packets from that client rather than forwarding them. This ensures mute is enforced regardless of client behaviour.

### Hand-Rolled Signaling

Nothing above requires a WebRTC library; the SDP is text and the transactions are ordinary Hotline transactions. A client that builds its SDP by hand — because no WebRTC stack exists for its platform — needs to know the following, all of which a library would otherwise hide:

- **The server is ICE-lite**, so the client is always the controlling agent and full ICE is unnecessary. One STUN Binding Request to the server's host candidate, carrying `USERNAME` (`<server-ufrag>:<client-ufrag>`), `MESSAGE-INTEGRITY` keyed with the **server's** `ice-pwd` (RFC 8445 §7.2.2: a request is keyed with the recipient's password), `ICE-CONTROLLING`, `PRIORITY`, `USE-CANDIDATE` and `FINGERPRINT`, nominates the pair. The server answers with a Binding Success carrying `XOR-MAPPED-ADDRESS`. The request is repeated as a keepalive.
- **The server's candidates are inline** in the offer, followed by `a=end-of-candidates`. A hand-rolled client can take the address from the first `a=candidate` line and never handle Voice ICE Candidate (604) at all; it need not send any 604 of its own if the server can learn its address from the STUN request, which an ICE-lite server can.
- **Audio is attributed by SSRC.** Nothing in an RTP packet carries the `mid`; the mapping from packet to user goes packet SSRC → `a=ssrc` line → `a=mid` → user, using the most recent offer. See [Track-to-User Mapping](#track-to-user-mapping).
- **The answer must mirror the offer**: one `m=` section per offered section, in the same order, with the same `a=mid`, and with a direction that is the complement of the offer's (`a=sendonly` answers `a=recvonly`; `a=recvonly` answers `a=sendonly`; `a=inactive` answers `a=inactive`). The `send` section MUST carry the client's microphone `a=ssrc` (see [Send SSRC Declaration](#send-ssrc-declaration)).
- **What can be dropped**: `a=rtcp-fb`, `a=extmap`, `a=extmap-allow-mixed`, `a=rtcp-rsize` and `a=msid` from the offer need not appear in the answer. The `/1` channel suffix on `a=rtpmap:0 PCMU/8000/1` is optional either way.

With DTLS-SRTP the hard part is not the signaling but the transport that follows it; see [Implementation Notes](#implementation-notes) for what that costs on a constrained platform, and the [plain RTP transport](#plain-rtp-transport) for the alternative.

---

## Media Transport

### UDP Port

The server opens a UDP listener for WebRTC media. The port is configurable via `VoiceUDPPort`; the default is **base port + 4**, following the existing port convention:

| Port | Usage |
|---|---|
| Base port | Transactions (TCP) |
| Base port + 1 | File transfers (TCP) |
| Base port + 2 | HTTP tunneling — transactions (TCP) |
| Base port + 3 | HTTP tunneling — transfers (TCP) |
| **Base port + 4** | **Voice media (UDP)** |

A single UDP port handles all WebRTC sessions. The WebRTC stack demultiplexes connections using DTLS fingerprints and ICE credentials. Sessions on the [plain RTP transport](#plain-rtp-transport) share the same port: their STUN requests are told apart by ICE ufrag and their media by bound source address, so enabling that transport opens nothing new.

### WebRTC Session

Each voice participant has one WebRTC peer connection to the server. The peer connection carries:

- **One send track** — the client's microphone audio
- **N−1 receive tracks** — audio from every other participant in the room

When a new participant joins, the server renegotiates existing peer connections to add the new receive track. When a participant leaves, the track is removed and connections are renegotiated.

### Track-to-User Mapping

Clients need to know which receive track corresponds to which user (e.g. to display "User X is speaking" or to allow per-user volume control). The server communicates this mapping through **SDP `a=mid` attributes**.

The server assigns each media section a `mid` value in the format `user-{UID}`, where `{UID}` is the decimal user ID. For example, a room with users 5, 12, and 23 would produce a server SDP offer sent to user 5 with:

```
m=audio 9 UDP/TLS/RTP/SAVPF 0
a=mid:user-12
a=sendonly
...

m=audio 9 UDP/TLS/RTP/SAVPF 0
a=mid:user-23
a=sendonly
...

m=audio 9 UDP/TLS/RTP/SAVPF 0
a=mid:send
a=recvonly
...
```

(Directions are from the server's — the offerer's — perspective: `a=sendonly` on sections whose audio the server forwards to the client, `a=recvonly` on the section carrying the client's microphone.)

| `mid` value | Meaning |
|---|---|
| `send` | The local client's send track |
| `user-{UID}` | Receive track carrying audio from user with that ID |

`{UID}` is the decimal string representation of the user's Hotline user ID (a `uint16`). Valid values are `1` through `65535`. Leading zeros MUST NOT be used (e.g. `user-5`, not `user-05`). User ID `0` is reserved and MUST NOT appear in a `mid` value.

Clients parse the `mid` labels to associate incoming audio tracks with users. The user IDs correspond to the standard Hotline user IDs visible in the chat room user list.

A `mid` is a property of the SDP, not of the packets: RTP packets carry no `mid` unless the RTP MID header extension (RFC 8843 §15) is negotiated, and this specification does not negotiate it. What a packet does carry is its SSRC, so **every `user-{UID}` section of the server's offer MUST declare, with `a=ssrc:<ssrc> cname:<cname>`, the SSRC the client will observe on packets carrying that user's audio**, and the server MUST stamp forwarded packets with exactly that SSRC — rewriting the source's own SSRC if they differ. A WebRTC library performs the SSRC → `mid` step internally; a client that reads RTP itself performs it from the offer. A renegotiation offer MAY change a section's declared SSRC (for instance when a user leaves and rejoins), and the client MUST use the mapping from the most recent offer it has answered.

When a participant leaves, the server sends a renegotiation offer (602) in which that user's media section is marked `a=inactive` (standard offer/answer direction semantics, [RFC 3264 §8.4](https://datatracker.ietf.org/doc/html/rfc3264#section-8.4)); the port remains `9`. The `m=` line remains in the SDP to preserve media section indexing — implementations MUST NOT delete `m=` lines from subsequent offers, as this would misalign `sdpMLineIndex` values.

A `mid` is never reassigned to a different user within a session: `user-{UID}` always carries the audio of user `{UID}`. If a user leaves and later rejoins the same room (with the same user ID), the server reactivates the existing inactive section — the same `mid` returns to `a=sendonly` in the renegotiation offer rather than a duplicate section being added. Clients MUST treat the `mid` and direction in each SDP offer as the authoritative mapping: a section is live only while its offered direction is `a=sendonly`. If a `mid` references a user ID not present in the most recent Voice Room Status (605) participant list, the client SHOULD accept the track but MAY treat it as inactive until the user appears in a subsequent status update.

Clients MUST tolerate media sections whose `mid` is neither `send` nor `user-{UID}`. Such sections can appear in renegotiation offers — for example, a server-side compatibility fallback creates one when a client's answer omits its [send SSRC declaration](#send-ssrc-declaration) — and future revisions of this specification may define additional labels. A client encountering an unrecognized `mid` MUST NOT map it to a user or play audio from it, MUST NOT reject the offer because of it, and MUST still mirror the section in its answer as required by RFC 3264 (answering it with the direction complement or `a=inactive` is acceptable).

### Renegotiation Flow

Renegotiation occurs when the participant list changes. The server MUST serialise renegotiations per peer — it MUST NOT send a new SDP offer to a client while a previous offer to that client is still awaiting an answer, as this results in undefined behaviour in most WebRTC stacks.

Serialisation is achieved by deferral: if the participant list changes while an offer to a client is outstanding, the server applies the track changes internally but withholds the new offer. When the client's answer (603) arrives, the server immediately issues a single follow-up offer (602) consolidating every change that accumulated in the meantime. Consequently, a client may receive a new offer directly after its answer is processed — clients MUST be prepared to run consecutive offer/answer cycles back to back, answering each offer in turn.

If a client nevertheless receives a new SDP offer (602) while it has not yet answered a previous offer, the client MUST discard the previous unanswered offer and process only the newest one. In practice, well-behaved servers will not trigger this condition due to the serialisation requirement above, but clients SHOULD handle it defensively.

The following sequence shows the concrete message flow when User B joins a room where User A is already in voice:

```
  Client A                   Server                    Client B
     │                          │                          │
     │                          │  ◄── Join Voice Room (600)
     │                          │         {ChatID: 42}     │
     │                          │                          │
     │                          │  ──► Join Reply (600)    │
     │                          │       {SDP offer,        │
     │                          │        codec: "PCMU",    │
     │                          │        participants: [A]}│
     │                          │                          │
     │                          │  ◄── Voice SDP Answer    │
     │                          │       (603) {SDP answer} │
     │                          │                          │
     │  ◄── Voice SDP Offer     │                          │
     │      (602) {updated SDP  │                          │
     │      with mid:user-B}    │                          │
     │                          │                          │
     │  ──► Voice SDP Answer    │                          │
     │      (603) {SDP answer}  │                          │
     │                          │                          │
     │  ◄── ICE Candidate (604) │  ──► ICE Candidate (604) │
     │  ──► ICE Candidate (604) │  ◄── ICE Candidate (604) │
     │                          │                          │
     │  ◄── Voice Room Status   │  ──► Voice Room Status   │
     │      (605) [A, B]        │      (605) [A, B]        │
     │                          │                          │
     │          ═══ Media flows between A ↔ Server ↔ B ═══ │
```

When User B leaves (or disconnects):

```
  Client A                   Server                    Client B
     │                          │                          │
     │                          │  ◄── Leave Voice Room    │
     │                          │       (601) {ChatID: 42} │
     │                          │                          │
     │  ◄── Voice SDP Offer     │  ──► Leave Reply (601)   │
     │      (602) {updated SDP  │                          │
     │      mid:user-B inactive}│                          │
     │                          │                          │
     │  ──► Voice SDP Answer    │                          │
     │      (603) {SDP answer}  │                          │
     │                          │                          │
     │  ◄── Voice Room Status   │                          │
     │      (605) [A]           │                          │
```

### Session Timeout and Failure

If the WebRTC session fails to establish (e.g. ICE connectivity check failure, DTLS handshake timeout), the following behaviour applies:

| Condition | Timeout | Action |
|---|---|---|
| No SDP answer received after Join Voice Room reply | 10 seconds | Server tears down the pending peer connection and sends Voice Room Status (605) removing the user from voice participants. |
| ICE connectivity checks fail (no valid pair found) | 30 seconds (WebRTC default) | The WebRTC stack reports failure. Server cleans up and sends Voice Room Status (605). |
| DTLS handshake failure | 10 seconds | Same as above. |
| Media timeout (no RTP/RTCP received from client) | 30 seconds | Server assumes the client's media path is dead. Tears down peer connection and sends Room Status (605) and Leave Voice Room notification. On the [plain RTP transport](#plain-rtp-transport), an authenticated STUN Binding Request also resets this timer. |

All timeouts SHOULD be measured using monotonic clocks to avoid issues with system clock adjustments. The specific timeout values above are recommendations — implementations MAY adjust them, but SHOULD NOT use values shorter than those listed.

Clients that detect a failed WebRTC session SHOULD update their UI to reflect that voice is no longer active and MAY present an option to retry (re-send Join Voice Room 600).

The Hotline TCP connection is **not** affected by WebRTC session failure — text chat continues normally.

### Stream Topology

```
Client A ──[send]──► Server ──[forward]──► Client B (receive track for A)
                          │──[forward]──► Client C (receive track for A)
                          └──[forward]──► Client D (receive track for A)

Client B ──[send]──► Server ──[forward]──► Client A (receive track for B)
                          │──[forward]──► Client C (receive track for B)
                          └──[forward]──► Client D (receive track for B)

(and so on for each participant)
```

### DTLS and SRTP

By default all media is encrypted: WebRTC mandates DTLS for key exchange and SRTP for media encryption, and any conformant WebRTC stack handles both without configuration. The DTLS fingerprint is exchanged in the SDP offer/answer, binding the media encryption to the signaling session.

The one exception is the [plain RTP transport](#plain-rtp-transport) below, which a server offers only when its operator has opted in and only to clients that ask for it. A server MUST NOT offer, and a client MUST NOT accept, unencrypted media in any other circumstance.

For a client that implements DTLS-SRTP by hand rather than through a WebRTC library, the requirements that matter in practice are: DTLS 1.2 with an ECDHE key exchange (X25519 is universally supported and by far the cheapest), a self-signed certificate (which may be generated once and stored; its fingerprint goes in the answer), and the `use_srtp` extension (RFC 5764) offering at least `SRTP_AES128_CM_HMAC_SHA1_80`. Servers MUST accept that SRTP protection profile and an X25519 key exchange, so that a minimal client has one known-good configuration to target.

### Plain RTP Transport

Some platforms have no TLS implementation and no cycles to spare for one: a 68k Macintosh cannot complete an ECDHE handshake in reasonable time, and cannot run AES and HMAC-SHA1 on fifty packets a second. For such clients this specification defines an alternative transport in which **everything except the encryption is unchanged**: the same transactions, the same SDP offer/answer, the same `mid` labelling and renegotiation rules, the same ICE credentials, the same UDP port. What differs is that the media sections are `RTP/AVP` instead of `UDP/TLS/RTP/SAVPF`, there is no DTLS handshake, and RTP travels in the clear.

**Availability.** The plain transport is an operator opt-in (`VoiceAllowPlainRTP`, default off). It is not advertised at login; a client simply asks for it in Join Voice Room (600) and is refused if it is not available. Clients that do not ask are unaffected, and a room may hold plain and DTLS-SRTP participants together — the server bridges the two, and a participant cannot tell which transport the others use.

**Requesting it.** The client sets `DATA_VOICE_TRANSPORT` (`0x01FB`) to `1` in Join Voice Room (600). A successful reply echoes the field with the transport in use, which is always the one requested; a transport the server will not provide is refused with an error. The server MUST NOT downgrade a DTLS request to plain, nor upgrade a plain request to DTLS. A client that receives an error for a plain request MAY tell the user that the server does not allow unencrypted voice; it MUST NOT retry with `0` unless it can actually do DTLS-SRTP.

**The offer** is built as in [SDP Format](#sdp-format), with these differences:

| Attribute | DTLS-SRTP | Plain RTP |
|---|---|---|
| `m=audio` transport | `UDP/TLS/RTP/SAVPF` | `RTP/AVP` |
| `m=audio` port and `c=` address | placeholder `9`, `0.0.0.0` | the server's real UDP port and primary address, so a client that ignores candidates still finds the server |
| `a=fingerprint`, `a=setup` | present | absent |
| `a=ice-lite`, `a=ice-ufrag`, `a=ice-pwd`, `a=candidate`, `a=end-of-candidates` | present | present, on every section |
| `a=ssrc` on `user-{UID}` sections | present | present |
| Voice ICE Candidate (604) | trickled after the offer | never sent by the server; candidates are complete in the offer |

```
v=0
o=- 1234567890 1 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-lite
a=group:BUNDLE send user-12
a=msid-semantic: WMS

m=audio 5504 RTP/AVP 0
c=IN IP4 198.51.100.7
a=mid:send
a=rtpmap:0 PCMU/8000
a=recvonly
a=rtcp-mux
a=ice-ufrag:WHbXqYAVTQjmsurm
a=ice-pwd:PiVrDKCAIeBcHEAbEvmUzzjQGDuwSEnN
a=candidate:1 1 udp 2130706431 198.51.100.7 5504 typ host
a=end-of-candidates

m=audio 5504 RTP/AVP 0
c=IN IP4 198.51.100.7
a=mid:user-12
a=rtpmap:0 PCMU/8000
a=sendonly
a=rtcp-mux
a=ice-ufrag:WHbXqYAVTQjmsurm
a=ice-pwd:PiVrDKCAIeBcHEAbEvmUzzjQGDuwSEnN
a=ssrc:1862706922 cname:user-12
a=candidate:1 1 udp 2130706431 198.51.100.7 5504 typ host
a=end-of-candidates
```

**The answer** mirrors the offer as described in [Hand-Rolled Signaling](#hand-rolled-signaling), with `RTP/AVP` on each `m=` line and no `a=fingerprint` or `a=setup`. The server needs only two things from it: that every section is answered, and that the `send` section accepts payload type 0. The client's `a=ice-ufrag`/`a=ice-pwd` are not used (the server never sends connectivity checks) and MAY be omitted; the `send` section's `a=ssrc` is informational on this transport and SHOULD still be present. Renegotiation offers (602) MUST be answered like any other; the [serialisation rule](#renegotiation-flow) applies unchanged.

**Binding.** Before the server will send the client anything, or attribute anything to it, the client MUST bind its address by sending a STUN Binding Request (RFC 8489) to the server's candidate address, exactly as in the [hand-rolled ICE](#hand-rolled-signaling) case:

| Attribute | Value |
|---|---|
| `USERNAME` | `<server ice-ufrag>:<client ice-ufrag>` — the client's own ufrag may be any string |
| `MESSAGE-INTEGRITY` | HMAC-SHA1 keyed with the **server's** `ice-pwd` from the offer |
| `FINGERPRINT` | CRC-32 of the message XOR `0x5354554e`; RECOMMENDED and validated if present |
| `ICE-CONTROLLING`, `PRIORITY`, `USE-CANDIDATE` | MAY be present; the server does not require them |

The server validates `MESSAGE-INTEGRITY` (and `FINGERPRINT` if present), records the request's source address as the client's media address, and replies with a Binding Success carrying `XOR-MAPPED-ADDRESS`. A request that fails validation is dropped without reply. Later authenticated requests from a different address re-bind the client (NAT rebinding). The server MUST NOT accept media from, or send media to, an address that has not bound.

The client MUST repeat the Binding Request at least every **15 seconds** for the life of the session. This keeps NAT mappings open and, because the server treats an authenticated request as liveness, keeps a muted client — which sends no RTP — from being timed out as dead (see [Session Timeout and Failure](#session-timeout-and-failure)).

**Media.** RTP is as in [RTP and RTCP](#rtp-and-rtcp): PCMU, payload type 0, 160-byte payloads, no SRTP. Inbound packets are attributed to the client by source address, so the SSRC the client sends with is its own affair; the server rewrites it before forwarding. Outbound packets carry the SSRC declared for that user's section in the most recent offer, and are sent to the bound address. The server MUST forward only payload type 0 from a plain client and MUST discard anything else. RTCP is OPTIONAL in both directions on this transport: a client MAY send Receiver Reports and the server MAY ignore them; the server sends none. Server-side mute enforcement applies exactly as for DTLS-SRTP participants.

**Not available on this transport.** The [video extension](Capabilities-Video.md) is layered on the WebRTC peer connection, which a plain-transport session does not have; a server MUST reject Video Start (607) from a participant on the plain transport.

**Security considerations.** Audio on this transport is readable and forgeable by anyone on the path — the same standing that Hotline's chat, private messages and (obfuscated, not encrypted) passwords have on the TCP connection, which is why an operator who accepts the one may reasonably accept the other. The STUN binding proves possession of the `ice-pwd`, which travels over that same TCP session, so it protects against off-path injection (an attacker who cannot see the offer cannot bind as the client) and not against an on-path attacker. Operators who run the base protocol over TLS or HOPE should note that voice on the plain transport is still unencrypted. Operators who cannot accept unencrypted audio simply leave `VoiceAllowPlainRTP` off, which is the default.

### RTP and RTCP

**RTP packet format** for PCMU audio follows [RFC 3550](https://datatracker.ietf.org/doc/html/rfc3550) with static payload type 0:

| Field | Size | Value | Notes |
|---|---|---|---|
| Version (V) | 2 bits | `2` | Always RTP version 2 |
| Padding (P) | 1 bit | `0` | No padding |
| Extension (X) | 1 bit | `0` or `1` | Header extensions may be present |
| CSRC Count (CC) | 4 bits | `0` | SFU does not add contributing sources |
| Marker (M) | 1 bit | varies | Set on the first packet after a silence period; generation is handled by the WebRTC stack's codec layer |
| Payload Type (PT) | 7 bits | `0` | Static PT for PCMU (RFC 3551) |
| Sequence Number | 16 bits | incrementing | Per-stream, wraps at 65535 |
| Timestamp | 32 bits | incrementing | 8000 Hz clock; increments by 160 per 20 ms frame |
| SSRC | 32 bits | random | Synchronisation source identifier |
| Payload | 160 bytes | μ-law samples | 20 ms of audio at 8000 Hz |

The SFU forwards RTP payloads, sequence numbers and timestamps without modification. It **does** rewrite the SSRC: each forwarded copy carries the SSRC declared in the receiving client's offer for that user's section (see [Track-to-User Mapping](#track-to-user-mapping)), which is what makes the mapping stable and immune to collisions between sources. Sequence numbers and timestamps are continuous per source, so a receiver's jitter buffer sees a well-formed stream. In WebRTC stacks the SSRC-to-track binding is done by the library from the `a=ssrc` attributes; a client that reads RTP itself does the same lookup from the offer.

**RTCP** is multiplexed on the same port as RTP (`a=rtcp-mux` is mandatory). WebRTC stacks handle RTCP automatically, generating Sender Reports (SR), Receiver Reports (RR), and other feedback. Implementations SHOULD NOT suppress or filter RTCP — it is needed for jitter buffer adaptation, lip-sync (if video is added in the future), and connectivity keepalives.

The SFU SHOULD forward RTCP feedback between peers to support receiver-side quality adaptation. At minimum, the SFU MUST respond to RTCP as required to keep the DTLS/ICE session alive.

Detailed RTCP message handling (aggregation, filtering, Reduced-Size RTCP per RFC 5506) is left to the WebRTC stack. Implementors using a standards-compliant WebRTC library do not need to handle RTCP manually.

**RTP header extensions** are not required by this specification. Implementations MAY negotiate header extensions (e.g. `urn:ietf:params:rtp-hdrext:ssrc-audio-level` per RFC 6464 for speaker activity indicators) via SDP `a=extmap`, but interoperability MUST NOT depend on their presence.

---

## Room Model

Voice is tied to Hotline's existing chat room model:

| Chat ID | Room | Notes |
|---|---|---|
| `0` | Public chat | The main chat room. Voice here acts as a "lobby" voice channel. |
| `> 0` | Private chat room | Created via standard Hotline chat room transactions. |

A user can be in text chat without being in voice (default). Voice participation is opt-in per room via Join Voice Room (600).

A user may only be in voice in **one room at a time**. Joining voice in a second room implicitly leaves voice in the first. This simplifies client UI and server resource management.

When an implicit leave occurs (user joins voice in room B while in voice in room A), the server MUST complete the teardown of room A before proceeding with the join in room B:
1. Tears down the WebRTC peer connection for room A.
2. Sends a **Voice Room Status (605)** notification to all remaining voice participants in room A, with the user removed from the participant list.
3. Proceeds with the Join Voice Room flow for room B as normal.

If the join in room B fails (e.g. the room is full), the user is left in no voice room. The server MUST NOT attempt to re-join the user to room A automatically — the client MAY retry by sending a new Join Voice Room (600) for room A.

---

## Access Privileges

Hotline uses a fine-grained access privilege bitmask (see the base protocol's Access Privileges section). Voice chat introduces one new privilege bit:

| Bit | Name | Description |
|---|---|---|
| 55 | `accessVoiceChat` | User may join voice chat rooms |

Bit 55 is the first available bit after the GLoarbLine extended privileges (bits 41–54). See the base protocol's Access Privileges section for the full bit map.

### Room Membership

A voice room is addressed by a bare `DATA_CHATID` and carries no membership of its own, so **the privilege bit alone is not an authorization model**. A server that checks only `accessVoiceChat` will let any voice-capable client join any room whose ID it can name or guess. Servers MUST additionally enforce:

| Room | Who may join |
|---|---|
| Public chat (`DATA_CHATID` = `0`) | Any client holding `accessVoiceChat` |
| A private chat's room | That chat's current members |
| A messenger call's room | Only the parties to that call — see [Instant Messaging](Capabilities-Messaging.md#call-invite-818) |

The last row matters even for servers that do not implement the messaging extension, because it constrains the ID space: a call's room ID and a private chat's ID are drawn from the same 32-bit space, so **room identifiers MUST be allocated by the server, from a cryptographically secure random source, and checked against every room already in use**. A predictable or client-chosen identifier lets a client name a room it was never invited to.

This is not hypothetical. A classic Hotline client with voice support has no concept of the messaging extension at all, and without these rules could join a private call between two messenger users simply by naming its identifier.

**Behaviour:**
- If `accessVoiceChat` is **not set**, the server rejects Join Voice Room (600) with an error: `"You are not allowed to join voice chat."`
- The privilege says a user may join voice rooms; it does **not** say *which*. See [Room Membership](#room-membership).
- The `CAPABILITY_VOICE` bit is still echoed in the login reply regardless of the user's privilege — the capability indicates server support, not user permission. This allows clients to display voice UI in a disabled state with a tooltip ("Voice chat requires permission") rather than hiding it entirely.
- Administrators and operators should have `accessVoiceChat` set by default.
- Servers that do not implement access privilege checking for voice may treat the bit as always set (allowing all users).

---

## Bandwidth Considerations

PCMU at 64 kbps (fixed):

| Participants | Upload per client | Download per client | Server total forwarding |
|---|---|---|---|
| 2 | 64 kbps | 64 kbps | 128 kbps |
| 5 | 64 kbps | 256 kbps | 1.28 Mbps |
| 10 | 64 kbps | 576 kbps | 5.76 Mbps |
| 16 (max default) | 64 kbps | 960 kbps | 15.36 Mbps |

PCMU does not support DTX, so silent participants still consume full bandwidth. For the small room sizes typical of Hotline (2–5 participants), this is well within modern network capacity. Operators hosting larger rooms should ensure adequate upstream bandwidth.

---

## Client Behaviour

- **Capability advertisement:** Set `CAPABILITY_VOICE` during login. If the server does not echo it, disable all voice UI.
- **Room join:** When the user clicks a voice button in a chat room, send Join Voice Room (600). On success, initialise the WebRTC peer connection using the SDP offer from the reply.
- **Audio capture:** Begin capturing microphone audio only after the WebRTC session is established. Audio MUST be captured at 8000 Hz, mono (1 channel), 16-bit signed linear PCM, encoded to G.711 μ-law before packetisation. Each RTP packet carries one 20 ms frame (160 samples = 160 μ-law bytes). Most WebRTC stacks handle resampling and encoding internally — if the platform's audio API provides audio at a different sample rate, the WebRTC codec layer will resample to 8000 Hz automatically.
- **Mute default:** Clients SHOULD join **muted by default** to avoid unexpected audio. The user explicitly unmutes.
- **UI indicators:** Display speaker/muted icons next to users in the chat room user list based on Voice Room Status (605) notifications.
- **Graceful degradation:** If the user's system has no microphone, the client may still join voice as a listen-only participant (send track is simply silent/absent).
- **Push-to-talk (PTT):** PTT is a client-side UX mode, not a protocol feature. The client sends Voice Mute (606) with `DATA_VOICE_MUTED = 0` when the PTT key is pressed and `DATA_VOICE_MUTED = 1` when released. No additional transactions are required. Clients may offer both PTT and open-mic modes as a user preference.
- **Leave on disconnect:** If the client loses connection, voice is cleaned up server-side automatically.
- **Early audio:** Clients MUST NOT send RTP packets before the WebRTC session is fully established (ICE connected + DTLS handshake complete). Packets sent before this point will be dropped by the transport layer. The WebRTC stack signals readiness via a connection state callback (e.g. `ICEConnectionStateConnected` or `PeerConnectionStateConnected`). On the plain RTP transport, readiness is the Binding Success response to the client's STUN request.
- **Transport choice:** A client that can do DTLS-SRTP MUST use it and MUST NOT request the plain transport merely because it is simpler. The plain transport exists for platforms on which DTLS-SRTP is not implementable, not as a convenience.

---

## Server Behaviour

- **SFU lifecycle:** The server creates a WebRTC peer connection per voice participant. When the participant leaves or disconnects, the peer connection is closed and resources freed.
- **Track management:** When a participant joins, add a receive track for them on every existing participant's peer connection (renegotiation). When they leave, remove the track.
- **Mute enforcement:** When a client's mute flag is set, the server MUST discard their incoming RTP packets rather than forwarding them. Do not rely on the client to stop sending. The server identifies the source of each RTP stream by the PeerConnection it arrives on (or, on the plain RTP transport, by the bound source address) — each participant has exactly one, so no SSRC-to-user mapping is required for attributing packets to users. (This does not relax the [send SSRC declaration](#send-ssrc-declaration) requirement: within a PeerConnection, the server's WebRTC stack still needs the answer's `a=ssrc` line to route inbound packets to the microphone track at all.) The server simply checks the mute state of the user associated with the receiving PeerConnection before forwarding each packet.
- **Room cleanup:** When the last voice participant leaves a room, tear down all SFU state for that room.
- **Resource limits:** Enforce `VoiceMaxPerRoom`. Reject Join Voice Room with an error if the limit is reached.
- **Access control:** The server MUST check that the client has permission to be in the chat room before allowing voice join. If a user is kicked from a chat room, their voice session MUST also be terminated.
- **No transcoding:** The server never decodes or re-encodes audio. It forwards RTP packets as-is.
- **Codec enforcement:** The server offers only PCMU. Clients that do not support PCMU cannot join voice. Since PCMU is mandatory-to-implement in all WebRTC stacks (RFC 7874), this is not expected to be a compatibility issue. If a client's SDP answer does not include PCMU (payload type 0), the server MUST reject the answer and tear down the pending peer connection.
- **Mute toggle debouncing:** Clients using push-to-talk may produce rapid mute/unmute transitions (key bouncing). The server SHOULD coalesce Voice Room Status (605) notifications rather than broadcasting every toggle — a debounce window of ~100 ms is RECOMMENDED before emitting a status update to other participants.

---

## Implementation Notes

- **Go implementations:** [Pion WebRTC](https://github.com/pion/webrtc) (`github.com/pion/webrtc/v4`) is a pure-Go WebRTC stack with no CGo dependencies, suitable for both server and client SFU implementations.
- **Other languages:** Any RFC-compliant WebRTC library can implement this extension. Notable options include [libwebrtc](https://webrtc.googlesource.com/src/) (C++, used in Chromium and Electron), [webrtc-rs](https://github.com/webrtc-rs/webrtc) (Rust), and the browser's built-in `RTCPeerConnection` API (JavaScript).
- **Dependency consideration:** WebRTC is a non-trivial dependency. Servers that do not need voice can build without it if the voice subsystem is behind a build tag or compile-time flag.
- **Port documentation:** Operators should be informed that enabling voice requires UDP port access. Firewall documentation should be updated.
- **Logging:** Voice join/leave events SHOULD be logged. Audio content MUST NOT be logged.
- **Metrics:** Track active voice sessions, participants per room, and bandwidth usage via the existing metrics endpoint.
- **Client implementation:** Client-side audio capture and playback (e.g. via PortAudio, miniaudio, or platform APIs) is outside the scope of this protocol document.
- **Constrained clients without a WebRTC library:** the signaling is plain text over the existing connection and needs nothing beyond string handling; see [Hand-Rolled Signaling](#hand-rolled-signaling). The transport is where the cost is. On the plain RTP transport the whole media layer is a UDP socket, one STUN Binding Request (HMAC-SHA1 and CRC-32, a few hundred lines of C), RTP framing and G.711. With DTLS-SRTP the additional requirements are a DTLS 1.2 client with an X25519 key exchange, a self-signed certificate and one signature each way per join, and per-packet AES-128-CTR plus HMAC-SHA1 in both directions — roughly 100 packets a second in a two-person room. Portable C implementations exist (mbedTLS provides DTLS and the `use_srtp` extension; libsrtp provides the packet protection), and are the realistic path for a PowerPC client; on a 68k the per-packet cost is marginal on a 68040 and prohibitive below it, which is what the plain transport is for.
