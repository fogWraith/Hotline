# Voice Chat Extension

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document describes the voice chat extension to the Hotline protocol. It adds real-time voice communication using a server-side SFU (Selective Forwarding Unit) architecture, where all media routes through the server. Clients negotiate voice support during login, establish WebRTC peer connections to the server, and the server forwards audio streams between participants in the same chat room.

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
- [Media Transport](#media-transport)
  - [UDP Port](#udp-port)
  - [WebRTC Session](#webrtc-session)
  - [Track-to-User Mapping](#track-to-user-mapping)
  - [Renegotiation Flow](#renegotiation-flow)
  - [Session Timeout and Failure](#session-timeout-and-failure)
  - [Stream Topology](#stream-topology)
  - [DTLS and SRTP](#dtls-and-srtp)
  - [RTP and RTCP](#rtp-and-rtcp)
- [Room Model](#room-model)
- [Access Privileges](#access-privileges)
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
- `a=ice-ufrag` and `a=ice-pwd` for ICE authentication
- `a=group:BUNDLE` — all media sections MUST be bundled over a single transport (RFC 8843)
- `a=rtcp-mux` — RTCP MUST be multiplexed on the same port as RTP (RFC 5761)
- `a=setup:actpass` on the offer, `a=setup:active` on the answer — DTLS role negotiation (RFC 8842)
- `a=sendonly` or `a=recvonly` direction attribute on each media section

The `m=audio` line uses port `9`, which is the standard placeholder port in bundled WebRTC SDP (RFC 8843 §9.3). Port `9` has no transport-layer significance — actual media transport uses the ICE candidate addresses. Implementations MUST NOT attempt to connect to port 9.

#### Annotated SDP Offer Example

The following is a complete SDP offer the server would send to User 5 joining a room where users 12 and 23 are already in voice. Annotations (lines starting with `#`) are not part of the SDP.

```
# Session-level attributes
v=0
o=- 1234567890 1 IN IP4 0.0.0.0
s=-
t=0 0
a=group:BUNDLE user-12 user-23 send
a=msid-semantic: WMS

# Receive track for user 12
m=audio 9 UDP/TLS/RTP/SAVPF 0
c=IN IP4 0.0.0.0
a=mid:user-12
a=rtpmap:0 PCMU/8000
a=recvonly
a=rtcp-mux
a=setup:actpass
a=ice-ufrag:srvr
a=ice-pwd:servericepasswordvalue1234
a=fingerprint:sha-256 AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99

# Receive track for user 23
m=audio 9 UDP/TLS/RTP/SAVPF 0
c=IN IP4 0.0.0.0
a=mid:user-23
a=rtpmap:0 PCMU/8000
a=recvonly
a=rtcp-mux
a=setup:actpass
a=ice-ufrag:srvr
a=ice-pwd:servericepasswordvalue1234
a=fingerprint:sha-256 AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99

# Send track for the joining client (user 5)
m=audio 9 UDP/TLS/RTP/SAVPF 0
c=IN IP4 0.0.0.0
a=mid:send
a=rtpmap:0 PCMU/8000
a=sendonly
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

# Accept send track
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
```

Key differences from the offer: `a=setup:active` (the answerer takes the DTLS client role), and the client's own ICE credentials and DTLS fingerprint replace the server's.

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

**Server Reply (success):**

| Field | ID | Type | Notes |
|---|---|---|---|
| Chat ID | 114 | UInt32 | Echoed back |
| Voice SDP | `0x01F5` | String | Server's SDP offer |
| Voice Codec | `0x01F7` | String | Room's active codec (e.g. `"PCMU"`) |
| Voice Participants | `0x01F9` | Binary | Current voice participants |

**Server Reply (error):**

Standard error reply with `DATA_ERROR_TEXT` (field 100). Error strings are human-readable and not intended for programmatic parsing. Clients SHOULD display the error text to the user as-is. Common error conditions include:
- Room is full (`VoiceMaxPerRoom` exceeded)
- Voice is disabled on the server
- Insufficient privileges (`accessVoiceChat` not set)

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
- When room conditions change and renegotiation is needed (e.g. codec change)

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

A single UDP port handles all WebRTC sessions. The WebRTC stack demultiplexes connections using DTLS fingerprints and ICE credentials.

### WebRTC Session

Each voice participant has one WebRTC peer connection to the server. The peer connection carries:

- **One send track** — the client's microphone audio
- **N−1 receive tracks** — audio from every other participant in the room

When a new participant joins, the server renegotiates existing peer connections to add the new receive track. When a participant leaves, the track is removed and connections are renegotiated.

### Track-to-User Mapping

Clients need to know which receive track corresponds to which user (e.g. to display "User X is speaking" or to allow per-user volume control). The server communicates this mapping through **SDP `a=mid` attributes**.

The server assigns each media section a `mid` value in the format `user-{UID}`, where `{UID}` is the decimal user ID. For example, a room with users 5, 12, and 23 would produce an SDP offer (from the perspective of user 5) with:

```
m=audio 9 UDP/TLS/RTP/SAVPF 0
a=mid:user-12
a=recvonly
...

m=audio 9 UDP/TLS/RTP/SAVPF 0
a=mid:user-23
a=recvonly
...

m=audio 9 UDP/TLS/RTP/SAVPF 0
a=mid:send
a=sendonly
...
```

| `mid` value | Meaning |
|---|---|
| `send` | The local client's send track |
| `user-{UID}` | Receive track carrying audio from user with that ID |

`{UID}` is the decimal string representation of the user's Hotline user ID (a `uint16`). Valid values are `1` through `65535`. Leading zeros MUST NOT be used (e.g. `user-5`, not `user-05`). User ID `0` is reserved and MUST NOT appear in a `mid` value.

Clients parse the `mid` labels to associate incoming audio tracks with users. The user IDs correspond to the standard Hotline user IDs visible in the chat room user list.

When a participant leaves, the server sends a renegotiation offer (602) that disables the media section for that user's `mid` by setting its port to `0` (e.g. `m=audio 0 UDP/TLS/RTP/SAVPF 0`). This follows SDP recycling conventions per [RFC 8829 §5.2.2](https://datatracker.ietf.org/doc/html/rfc8829#section-5.2.2). The `m=` line remains in the SDP to preserve media section indexing — implementations MUST NOT delete `m=` lines from subsequent offers, as this would misalign `sdpMLineIndex` values. Future renegotiations MAY reuse a disabled slot (port `0`) for a new participant by assigning a new `mid` and restoring the port to `9`.

When a media section slot is reused, the server MUST assign a new `mid` value (e.g. `user-42` replacing the disabled `user-23`). Clients MUST treat the `mid` in each SDP offer as the authoritative mapping and MUST NOT cache mid-to-user associations across renegotiations. If a `mid` references a user ID not present in the most recent Voice Room Status (605) participant list, the client SHOULD accept the track but MAY treat it as inactive until the user appears in a subsequent status update.

### Renegotiation Flow

Renegotiation occurs when the participant list changes. The server MUST serialise renegotiations — if users B and C join simultaneously, the server completes the full offer/answer cycle for B before beginning renegotiation for C. Sending a new SDP offer while a previous offer is awaiting an answer results in undefined behaviour in most WebRTC stacks and MUST be avoided.

If a client receives a new SDP offer (602) while it has not yet answered a previous offer, the client MUST discard the previous unanswered offer and process only the newest one. In practice, well-behaved servers will not trigger this condition due to the serialisation requirement above, but clients SHOULD handle it defensively.

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
     │      mid:user-B port=0}  │                          │
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
| Media timeout (no RTP/RTCP received from client) | 30 seconds | Server assumes the client's media path is dead. Tears down peer connection and sends Room Status (605) and Leave Voice Room notification. |

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

All media MUST be encrypted. WebRTC mandates DTLS for key exchange and SRTP for media encryption. This is handled automatically by any conformant WebRTC stack — no additional configuration is needed.

The DTLS fingerprint is exchanged in the SDP offer/answer, binding the media encryption to the signaling session.

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

The SFU forwards RTP packets without modification — it does not rewrite SSRCs, sequence numbers, or timestamps. SSRC collisions are theoretically possible but unlikely in small rooms. In WebRTC stacks, SSRC-to-track binding is managed internally by the library (via SDP `a=ssrc` attributes and BUNDLE demultiplexing); implementors do not need to handle SSRC collisions manually. If using a raw RTP implementation without WebRTC, consult RFC 3550 §8.2 for collision resolution procedures.

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

**Behaviour:**
- If `accessVoiceChat` is **not set**, the server rejects Join Voice Room (600) with an error: `"You are not allowed to join voice chat."`
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
- **Early audio:** Clients MUST NOT send RTP packets before the WebRTC session is fully established (ICE connected + DTLS handshake complete). Packets sent before this point will be dropped by the transport layer. The WebRTC stack signals readiness via a connection state callback (e.g. `ICEConnectionStateConnected` or `PeerConnectionStateConnected`).

---

## Server Behaviour

- **SFU lifecycle:** The server creates a WebRTC peer connection per voice participant. When the participant leaves or disconnects, the peer connection is closed and resources freed.
- **Track management:** When a participant joins, add a receive track for them on every existing participant's peer connection (renegotiation). When they leave, remove the track.
- **Mute enforcement:** When a client's mute flag is set, the server MUST discard their incoming RTP packets rather than forwarding them. Do not rely on the client to stop sending. The server identifies the source of each RTP stream by the PeerConnection it arrives on — each participant has exactly one PeerConnection, so no SSRC-to-user mapping is required. The server simply checks the mute state of the user associated with the receiving PeerConnection before forwarding each packet.
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
