# Video Chat Extension

> **Author:** Misha Nasledov ([@mishan](https://github.com/mishan))

> **Status:** Accepted, Implemented by the reference server, Janus, from 2.0.17.

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document describes the video chat extension to the Hotline protocol. It adds camera video and screen sharing to the voice chat rooms defined by the [Voice Chat Extension](Capabilities-Voice.md), reusing that extension's SFU, peer connection, room model and SDP negotiation. A participant may publish a camera stream, a screen-share stream, both, or neither, while remaining an ordinary voice participant.

**Video is opt-in on both sides.** Joining a room is joining voice; no video is published until the participant asks, and no video is received until the participant asks for that stream in particular. A client that implements none of this document is a full member of a room in which others are using all of it, and a client that implements all of it still sends and receives nothing until its user decides otherwise.

This extension is **layered on** the voice extension, not parallel to it. It defines no new peer connection, no new UDP port, no new SDP transaction and no new ICE transaction. What it adds is a set of control transactions for starting, pausing and stopping video publications, a status notification, and the SDP conventions that let a client tell one video stream from another.

---

## Table of Contents

- [Relationship to the Voice Extension](#relationship-to-the-voice-extension)
- [Architecture](#architecture)
  - [One Peer Connection](#one-peer-connection)
  - [Publications and Slots](#publications-and-slots)
- [Compatibility and Negotiation](#compatibility-and-negotiation)
  - [Capability Bit](#capability-bit)
  - [Degrading to Voice](#degrading-to-voice)
  - [Publication Bits](#publication-bits)
  - [Plain RTP Transport](#plain-rtp-transport)
  - [Server Configuration](#server-configuration)
- [Codec Negotiation](#codec-negotiation)
  - [Supported Codecs](#supported-codecs)
  - [Codec ID Table](#codec-id-table)
  - [Payload Types](#payload-types)
- [Stream Model](#stream-model)
  - [Stream Kinds](#stream-kinds)
  - [Start, Pause, Stop](#start-pause-stop)
- [Signaling](#signaling)
  - [Overview](#overview)
  - [Transaction Types](#transaction-types)
  - [Data Objects](#data-objects)
  - [Video Start (607)](#video-start-607)
  - [Video Stop (608)](#video-stop-608)
  - [Video State (609)](#video-state-609)
  - [Video Subscribe (610)](#video-subscribe-610)
  - [Video Status (611)](#video-status-611)
- [Media Transport](#media-transport)
  - [Track-to-User Mapping](#track-to-user-mapping)
  - [Annotated SDP Offer Example](#annotated-sdp-offer-example)
  - [Send SSRC Declaration](#send-ssrc-declaration)
  - [Renegotiation](#renegotiation)
  - [Keyframes and RTCP Feedback](#keyframes-and-rtcp-feedback)
  - [Bitrate](#bitrate)
  - [Session Failure](#session-failure)
- [Room Model](#room-model)
- [Access Privileges](#access-privileges)
- [Bandwidth Considerations](#bandwidth-considerations)
- [Client Behaviour](#client-behaviour)
- [Server Behaviour](#server-behaviour)
- [Privacy and Abuse](#privacy-and-abuse)
- [Hotline-ng Binding](#hotline-ng-binding)
- [Implementation Notes](#implementation-notes)
- [Reserved for Future Revisions](#reserved-for-future-revisions)

---

## Relationship to the Voice Extension

Everything in [Capabilities-Voice.md](Capabilities-Voice.md) continues to apply unchanged. This extension inherits, and does not restate:

- the SFU architecture and the reasons for it;
- the single UDP media port (base port + 4);
- Join Voice Room (600) / Leave Voice Room (601) and the room lifecycle;
- Voice SDP Offer (602), Voice SDP Answer (603) and Voice ICE Candidate (604) — **video renegotiation uses these transactions**, because it is the same peer connection being renegotiated;
- the server-is-always-the-offerer rule and the per-peer renegotiation serialisation rule;
- BUNDLE, `rtcp-mux`, DTLS-SRTP, port 9, and the `a=inactive` convention for a departed or stopped stream;
- one room at a time, and the room-membership rules that constrain which room a client may name.

A client that has negotiated voice but not video is a full participant in a room where others use video. A server that implements voice but not video is unaffected by this document.

---

## Architecture

### One Peer Connection

Each participant has exactly one WebRTC peer connection to the server, established by Join Voice Room (600). Video adds media sections to that connection:

```
                    ┌──────────────────────────────────┐
                    │            Server SFU            │
   one peer         │                                  │
   connection ─────▶│  audio  ──▶ forward to room      │
   per participant  │  camera ──▶ forward to room      │
                    │  screen ──▶ forward to room      │
                    └──────────────────────────────────┘
                              one UDP port
```

A single ICE session, a single DTLS handshake and a single SRTP context carry audio, camera and screen for a participant. Implementations MUST NOT open a second peer connection for video.

The consequence for clients is that **video requires an established voice session**. Video Start (607) outside a voice room is an error. A participant who wishes to be seen but not heard joins voice muted; the voice extension already provides for a listen-only participant, and mute is already server-enforced.

### Publications and Slots

A **publication** is one outbound stream from one participant, of one kind, in one room. A participant may hold at most one publication per kind: one camera, one screen share. Publications are independent of one another and of the participant's audio.

The server maintains, per room, a bounded number of publication **slots** per kind. A publication occupies its slot from Video Start until Video Stop, disconnect, or the participant leaving voice. Pausing does **not** release a slot — a paused publication still occupies one, which is what makes "pause" cheap and "stop" meaningful.

A **subscription** is the receive-side counterpart: one participant's declared interest in one publication. Publishing announces a stream's existence to the room; it does not deliver it to anybody. A peer receives a publication only while it holds a subscription to it, and every participant begins with none.

The asymmetry is deliberate. A publication is a room-wide fact, bounded by slots and visible to everyone: in detail through [Video Status (611)](#video-status-611), and as a bare fact through the [publication bits](#publication-bits) of the voice participant list. A subscription is a private arrangement between one participant and the server: it is not announced, not bounded, and the publisher is never told who is watching. What it bounds instead is the receiver's own bandwidth, which is the resource that actually runs out first — a phone on a cellular connection in a room of eight cameras.

---

## Compatibility and Negotiation

### Capability Bit

| Bit | Mask | Name | Description |
|---|---|---|---|
| 10 | `0x0400` | `CAPABILITY_VIDEO` | Camera video and screen sharing in voice rooms |

Bit 10 is the next available bit after `CAPABILITY_MODERN_DATES` (bit 9). This bit is defined in the `DATA_CAPABILITIES` bitmask (field `0x01F0`); see [DATA_CAPABILITIES](Capabilities.md) for the general negotiation flow.

**Bit 10 depends on bit 2.** A server MUST NOT confirm `CAPABILITY_VIDEO` unless it also confirms `CAPABILITY_VOICE` in the same login reply, and a client MUST NOT set bit 10 without also setting bit 2. Video has no meaning without the voice room that carries it. (This mirrors the dependency of bit 8 on bit 6 in the messaging extension.)

**Negotiation:**

1. Client sets bits 2 and 10 in `DATA_CAPABILITIES` during Login (107).
2. Server checks its configuration. If voice and video are both enabled and the SFU supports video, it echoes both bits.
3. If the server echoes bit 2 but not bit 10, the session has voice and no video: the client MUST NOT display video UI and MUST NOT send video transactions.
4. When bit 10 is confirmed, the server SHOULD include one `DATA_VIDEO_LIMITS` field per supported stream kind in the login reply, so the client can configure its encoders before the first join rather than discovering limits by rejection.

As with voice, the capability is echoed regardless of the user's privilege bits: the capability states server support, the privilege states user permission (see [Access Privileges](#access-privileges)).

### Degrading to Voice

This is the rule that makes the extension safe to deploy into an existing room, and it is normative:

> **A server MUST NOT include a video media section in an SDP offer to a peer that has not subscribed to that publication, and MUST NOT forward video RTP to a peer that has not subscribed to it. A client that has not confirmed `CAPABILITY_VIDEO` can hold no subscriptions, and therefore receives no video sections and no video RTP, ever.**

The second sentence is a consequence of the first rather than a separate rule, which is the point: a voice-only client is not a special case the server must remember to exclude. It is simply a peer whose subscription set is permanently empty, and it takes the same code path as a video-capable client that has not subscribed to anything yet.

The voice extension already requires clients to tolerate media sections whose `mid` they do not recognise, so a stray video section in a voice-only client's offer would not break it outright. It would, however, cause that client's stack to allocate receive transceivers it can never render, and — far worse — invite the server to forward megabits of video to a client that will decode it and throw it away. Video is never a room-wide mode.

Three corollaries:

- The set of media sections in an offer, and therefore the mapping from `sdpMLineIndex` to `mid`, differs between peers in the same room and changes as each peer's subscriptions change. It already differed under the voice extension — each peer's offer omits its own receive section — but video widens the gap considerably. Clients MUST key on `mid`, never on section position.
- A room's video slot accounting is global to the room, not per subscriber. A screen share occupies the room's screen slot whether one participant is watching it or none.
- Starting a publication does not renegotiate the room. It renegotiates the publisher, to add the publisher's own send section, and notifies everyone else. Peers renegotiate individually, if and when they subscribe.

### Publication Bits

A voice-only participant receives nothing from this extension's transactions, but it is still in the frame: it should be able to tell that a camera or a screen share is live in the room it is sitting in. The voice extension's participant entry (`DATA_VOICE_PARTICIPANTS`, `0x01F9`) therefore carries two publication bits in its flags word, alongside the mute bit. The voice extension lists them with the same rules; this section is where they are defined:

| Bit | Mask | Meaning |
|---|---|---|
| 0 | `0x0001` | Muted (defined by the voice extension) |
| 1 | `0x0002` | The participant has a camera publication |
| 2 | `0x0004` | The participant has a screen publication |
| 3–15 | — | Reserved |

- Servers MUST send reserved bits as zero. Clients MUST ignore reserved bits, and MUST read mute from bit 0 alone.
- Bits 1 and 2 are set in **every** `DATA_VOICE_PARTICIPANTS` the server sends — the Join Voice Room (600) reply and every Voice Room Status (605) — whatever the receiving client negotiated. A client that joins mid-share learns of it from its join reply, without waiting for the next status. A server that does not implement this extension leaves them zero.
- A bit reflects whether the publication exists, not whether it is paused. A paused camera still holds its publication and its slot; pause state is carried by Video Status (611) alone.
- The server sends Voice Room Status (605) to the room whenever a publication starts or stops, including when a publisher leaves voice, is kicked, disconnects, or has a failed publication reaped. Pausing and resuming change no bit and send no 605.
- These notifications MAY go through the same debounce as the voice extension's mute notifications. A coalesced 605 MUST carry the room's state at the moment it is sent, so a debounce may delay a change but never lose one.

Video-capable clients get the same fact, with detail, from Video Status (611), and SHOULD prefer it. The bits exist for the participant who has no 611 to read.

### Plain RTP Transport

The voice extension's [plain RTP transport](Capabilities-Voice.md#plain-rtp-transport) has no peer connection, and so nothing to add a video section to. A server MUST refuse Video Start (607) and Video Subscribe (610) from a participant on the plain RTP transport. Such a participant is otherwise an ordinary voice-only peer: it receives no video, and still sees who is publishing through the [publication bits](#publication-bits).

### Server Configuration

Following the voice extension's example, using Janus's naming style:

| Setting | Type | Default | Description |
|---|---|---|---|
| `EnableVideo` | bool | `false` | Master switch. Requires `EnableVoice`. |
| `VideoMaxCamerasPerRoom` | int | `8` | Simultaneous camera publications per room |
| `VideoMaxScreensPerRoom` | int | `1` | Simultaneous screen-share publications per room |
| `VideoMaxWidth` / `VideoMaxHeight` | int | `1280` / `720` | Camera ceiling |
| `VideoMaxFPS` | int | `30` | Camera ceiling |
| `VideoMaxBitrate` | int | `1500000` | Camera ceiling, bits per second |
| `ScreenMaxWidth` / `ScreenMaxHeight` | int | `1920` / `1080` | Screen-share ceiling |
| `ScreenMaxFPS` | int | `15` | Screen-share ceiling |
| `ScreenMaxBitrate` | int | `2500000` | Screen-share ceiling, bits per second |

The camera default is deliberately lower than the screen default in resolution and higher in frame rate; see [Bandwidth Considerations](#bandwidth-considerations).

---

## Codec Negotiation

### Supported Codecs

| Codec | Clock Rate | Typical Bitrate | Use Case |
|---|---|---|---|
| **VP8** | 90000 Hz | 150 kbps – 2.5 Mbps | Universal codec, royalty-free, no profile negotiation |

VP8 is the only codec this revision defines, for the same reason the voice extension defines only PCMU: an SFU does not transcode, so every participant in a room must use the same codec, and the only safe choice is the one every stack has.

VP8 is mandatory-to-implement for WebRTC endpoints alongside H.264 ([RFC 7742](https://datatracker.ietf.org/doc/html/rfc7742)), its payload format is a single unambiguous specification ([RFC 7741](https://datatracker.ietf.org/doc/html/rfc7741)), and — decisively for an SFU — it has **no profile negotiation**. H.264 carries `profile-level-id` and `packetization-mode` in `a=fmtp`, and two endpoints can both "support H.264" while sharing no mutually decodable profile. With no transcoding, a room would then have to negotiate a common profile across every participant and re-negotiate it whenever anyone joined. VP8 has one profile and one packetization.

H.264 remains the obvious second codec for hardware-decode-constrained clients, and is reserved below rather than defined here.

### Codec ID Table

Video codec IDs are a **separate number space** from the voice extension's codec IDs. Codec ID 0 means PCMU in `DATA_VOICE_PARTICIPANTS` and VP8 in `DATA_VIDEO_PUBLISHERS`; the two fields are never interchangeable and implementations MUST NOT share a lookup table between them.

| Codec ID | Name | SDP Encoding Name | Clock Rate |
|---|---|---|---|
| 0 | VP8 | `VP8` | 90000 |
| 1 | *(reserved)* H.264 | `H264` | 90000 |
| 2 | *(reserved)* VP9 | `VP9` | 90000 |
| 3 | *(reserved)* AV1 | `AV1` | 90000 |
| 4–65535 | Reserved | — | — |

### Payload Types

VP8 has no static payload type, so this specification fixes the dynamic values the server offers. Since the server is always the offerer, no negotiation is required:

| PT | Format | Notes |
|---|---|---|
| 96 | `VP8/90000` | All video sections, both kinds |
| 97 | `rtx/90000`, `a=fmtp:97 apt=96` | OPTIONAL retransmission stream |

A client's answer MUST NOT renumber: any payload type it keeps uses the number the offer gave it. A client that does not implement RTX answers the section without PT 97; RTX is then not negotiated for that section, and lost packets reach it, if at all, as retransmitted originals (see [Keyframes and RTCP Feedback](#keyframes-and-rtcp-feedback)).

**Both stream kinds use payload type 96.** Camera and screen video are the same codec at the same payload type, distinguished by `mid` and SSRC and by nothing else. This has a direct consequence for [Send SSRC Declaration](#send-ssrc-declaration).

---

## Stream Model

### Stream Kinds

| Kind ID | Name | Meaning |
|---|---|---|
| 0 | *(invalid)* | MUST NOT appear on the wire |
| 1 | `VIDEO_KIND_CAMERA` | The participant's camera |
| 2 | `VIDEO_KIND_SCREEN` | A shared screen, window, or region |
| 3 | *(reserved)* | Screen audio — see [Reserved](#reserved-for-future-revisions) |
| 4–65535 | Reserved | — |

Kind `0` is deliberately invalid so that a zeroed or uninitialised field is caught rather than silently interpreted as a camera.

The distinction is semantic, not merely descriptive: clients lay out a screen share differently from a camera (large, aspect-preserved, often as the focus of the window), and encoders tune differently for it. Screen-share sections additionally carry `a=content:slides` ([RFC 4796](https://datatracker.ietf.org/doc/html/rfc4796)), so a client that ignores this extension's status notifications can still identify a screen share from the SDP alone.

### Start, Pause, Stop

Three operations, deliberately distinct, because they cost different amounts:

| Operation | Transaction | Renegotiates? | Slot |
|---|---|---|---|
| **Start** | Video Start (607) | The publisher only, to add its own send section | Claimed |
| **Pause / resume** | Video State (609) | **No** | Held |
| **Stop** | Video Stop (608) | The publisher, and any peer subscribed to it — sections go `a=inactive` | Released |

**Pause is to video what mute is to audio**, and for the same reason. Turning a camera off and on again is a frequent, casual act; if every toggle renegotiated every peer connection in the room, a room of eight would spend its time in offer/answer. A paused publication keeps its media section, keeps its slot, and keeps its `mid`; the server simply stops forwarding its RTP, exactly as it stops forwarding a muted participant's audio.

Pause MUST be enforced by the server discarding the publisher's inbound RTP, not by trusting the client to stop sending. Clients SHOULD nonetheless stop capturing while paused, both to save power and because a camera that is off should have its hardware indicator off.

Stop is for finishing: releasing a screen-share slot so someone else may take it, or ending a publication the client does not expect to resume soon.

---

## Signaling

### Overview

Video signaling uses the existing Hotline TCP connection and the existing voice transactions for everything to do with SDP and ICE. Five new transactions carry publication control and status.

All video transactions are only sent to or from clients that have confirmed `CAPABILITY_VIDEO`.

Transaction semantics follow the voice extension exactly: 607–610 are request/reply with a client-chosen task ID and the reply flag set on the response; 611 is a server-initiated notification sent with task ID `0` and the reply flag unset, dispatched by type rather than matched against a pending-request queue.

### Transaction Types

| ID | Name | Direction | Description |
|---|---|---|---|
| 607 | Video Start | Client → Server | Begin publishing a stream of a given kind |
| 608 | Video Stop | Client → Server | End a publication and release its slot |
| 609 | Video State | Client → Server | Pause or resume a publication |
| 610 | Video Subscribe | Client → Server | Declare which streams this client wishes to receive |
| 611 | Video Status | Server → Client | Notification of publications and their state |

Transaction IDs 607–611 continue the voice extension's block (600–606); **612–619 are reserved** for future revisions of this document. The 600 block as a whole is thereby the media-signalling block, which keeps it clear of the base protocol (101–355), the keepalive (500), voice (600–606), chat history (700–709), inline media (750–751) and GIF icons (1861–1864).

### Data Objects

| ID (hex) | Name | Type | Description |
|---|---|---|---|
| `0x0220` | `DATA_VIDEO_KIND` | UInt16 | Stream kind; see [Stream Kinds](#stream-kinds) |
| `0x0221` | `DATA_VIDEO_PAUSED` | UInt16 | 0 = live, 1 = paused |
| `0x0222` | `DATA_VIDEO_PUBLISHERS` | Binary | Packed array of publication entries |
| `0x0223` | `DATA_VIDEO_CODEC` | String | Active video codec name for the room, e.g. `"VP8"` |
| `0x0224` | `DATA_VIDEO_LIMITS` | Binary | Server limits for one stream kind |
| `0x0225` | `DATA_VIDEO_SUBSCRIPTIONS` | Binary | Packed array of the streams this client wishes to receive |

Field IDs `0x0220`–`0x023F` are reserved for this extension. `0x0226`–`0x023F` are unallocated; implementations MUST NOT use them for unrelated purposes. (The block begins at `0x0220` because `0x01F5`–`0x01F9` belong to voice, `0x01FA` to the large-file extension's resume digest, and `0x0201`–`0x021F` to inline media.)

**`DATA_VIDEO_PUBLISHERS`** is a packed binary structure, one entry per publication — a participant publishing both camera and screen produces two entries:

```
For each publication (8 bytes):
  [2] User ID    (big-endian uint16)
  [2] Kind       (big-endian uint16: see Stream Kinds)
  [2] Flags      (big-endian uint16: bit 0 = paused, bits 1-15 reserved)
  [2] Codec ID   (big-endian uint16: see Codec ID Table)
```

The total field length divided by 8 gives the publication count. A trailing partial entry MUST be ignored.

This is a **new field, not an extension of `DATA_VOICE_PARTICIPANTS`.** The voice participants blob is a 6-byte-stride array whose count is derived by dividing the field length by 6, and deployed voice clients do exactly that; widening the stride would silently misparse in every one of them. Audio state stays in `DATA_VOICE_PARTICIPANTS` and video state in `DATA_VIDEO_PUBLISHERS`, and a client correlates them by user ID.

**`DATA_VIDEO_SUBSCRIPTIONS`** is a packed binary structure naming the streams a client wishes to receive. It is the client's **complete** desired set, not a delta:

```
For each desired stream (4 bytes):
  [2] User ID    (big-endian uint16)
  [2] Kind       (big-endian uint16: see Stream Kinds)
```

The total field length divided by 4 gives the count; a trailing partial entry MUST be ignored. A zero-length field, or the field omitted entirely, means "no video at all" and is the state every participant starts in.

Declaring the whole set rather than toggling one stream at a time is what keeps the cost down: a client opening a room with four cameras subscribes to all four in one request and pays for one renegotiation, rather than four requests serialised behind one another by the renegotiation rule. It is also idempotent, which matters on a protocol where a client may have to re-establish state.

**`DATA_VIDEO_LIMITS`** describes the server's ceiling for one stream kind. The field is repeated, once per kind the server supports, in the Login (107) reply:

```
  [2] Kind         (big-endian uint16)
  [2] Max width    (big-endian uint16, pixels)
  [2] Max height   (big-endian uint16, pixels)
  [2] Max FPS      (big-endian uint16)
  [4] Max bitrate  (big-endian uint32, bits per second)
  [2] Max per room (big-endian uint16, publication slots)
  [2] Reserved     (MUST be zero)
```

Sixteen bytes in this revision. A parser MUST accept a longer field and ignore the excess, so that a later revision can append.

### Video Start (607)

**Client → Server**

| Field | ID | Type | Required | Notes |
|---|---|---|---|---|
| Chat ID | 114 | UInt32 | Yes | The room the client is in voice in |
| Video Kind | `0x0220` | UInt16 | Yes | Camera or screen |

**Server Reply (success):**

| Field | ID | Type | Notes |
|---|---|---|---|
| Chat ID | 114 | UInt32 | Echoed |
| Video Kind | `0x0220` | UInt16 | Echoed |
| Video Codec | `0x0223` | String | Room's active video codec |

**The reply carries no SDP offer.** This differs from Join Voice Room (600) deliberately. A renegotiation may already be outstanding toward this peer, and the voice extension forbids a second offer before the first is answered; the server therefore replies immediately with success and sends the offer as a Voice SDP Offer (602) when serialisation allows. The client MUST NOT wait for an offer before considering the request to have succeeded, and MUST NOT begin capturing until the resulting negotiation completes.

**Server Reply (error):** standard error reply with `DATA_ERROR_TEXT` (field 100), human-readable and not intended for programmatic parsing. Conditions include: the client is not in voice in that room; the client is on the voice extension's [plain RTP transport](#plain-rtp-transport); a publication of that kind already exists for this client; no slot is free for that kind (`VideoMaxCamerasPerRoom` / `VideoMaxScreensPerRoom`); video is disabled; the user lacks `AccessVideoChat`, or `AccessScreenShare` for a screen publication.

The server also sends **Video Status (611)** to every video-capable participant in the room, and **Voice Room Status (605)** to every participant, with the publisher's [publication bit](#publication-bits) now set.

### Video Stop (608)

**Client → Server**

| Field | ID | Type | Required | Notes |
|---|---|---|---|---|
| Chat ID | 114 | UInt32 | Yes | |
| Video Kind | `0x0220` | UInt16 | No | Omit to stop **all** of this client's publications in the room |

**Server Reply:** empty success reply.

The publication's media section becomes `a=inactive` in the next offer to each peer, its slot is released, and Video Status (611) and Voice Room Status (605) are sent to the room, the latter with the publisher's [publication bit](#publication-bits) cleared. Stopping a publication that does not exist is not an error; the operation is idempotent, because disconnect races make it so.

### Video State (609)

**Client → Server**

| Field | ID | Type | Required | Notes |
|---|---|---|---|---|
| Chat ID | 114 | UInt32 | Yes | |
| Video Kind | `0x0220` | UInt16 | Yes | |
| Video Paused | `0x0221` | UInt16 | Yes | 0 = resume, 1 = pause |

**Server Reply:** empty success reply.

No renegotiation occurs. The server sends Video Status (611) to the room — but no Voice Room Status (605), since pausing changes no [publication bit](#publication-bits) — and SHOULD coalesce rapid toggles with a debounce window of approximately 100 ms, as the voice extension recommends for mute.

On resume, the server MUST request a keyframe from the publisher before or as it resumes forwarding; see [Keyframes and RTCP Feedback](#keyframes-and-rtcp-feedback).

### Video Subscribe (610)

**Client → Server**

Declares the complete set of streams this client wishes to receive in a room. **This is the only way video is ever delivered to a client**: a participant that never sends this transaction never receives video, which is the default state on joining.

| Field | ID | Type | Required | Notes |
|---|---|---|---|---|
| Chat ID | 114 | UInt32 | Yes | |
| Video Subscriptions | `0x0225` | Binary | No | Complete desired set; omit or send empty to receive nothing |

**Server Reply:** empty success reply. A participant on the [plain RTP transport](#plain-rtp-transport) is refused with a standard error reply, as for Video Start.

The server diffs the requested set against the client's current subscriptions and, if they differ, sends **one** consolidated Voice SDP Offer (602) reflecting every change — subject to the usual serialisation rule, so the offer may arrive after an outstanding one is answered rather than immediately. As with Video Start, the reply does not carry the offer.

A newly subscribed stream gains a media section in that peer's offer; an unsubscribed one has its section set to `a=inactive`, keeping its `mid` for a later resubscribe. This is the same machinery the voice extension already uses for a departing participant, and it means that from a receiver's point of view unsubscribing is indistinguishable from the publisher having stopped. The server MUST request a keyframe from the publisher when a subscription is added; see [Keyframes and RTCP Feedback](#keyframes-and-rtcp-feedback).

Subscriptions are per room and are discarded on leaving it. A subscription to a stream that does not exist — a user who is not publishing that kind, or is not in the room — is not an error: the server retains it and activates it if that publication appears. This lets a client express a standing preference ("show me everyone") without racing the room's state, and without polling.

No notification is sent to the room. Who is watching whom is not published, to the publisher or to anyone else.

**Servers implementing `CAPABILITY_VIDEO` MUST implement this transaction.** It is not a bandwidth optimisation that a minimal server may skip; it is the delivery mechanism, and a server without it would deliver no video at all.

### Video Status (611)

**Server → Client (notification)**

Sent to every video-capable participant in a room whenever the set of publications or their paused state changes — start, stop, pause, resume, or a publisher leaving voice or disconnecting.

| Field | ID | Type | Notes |
|---|---|---|---|
| Chat ID | 114 | UInt32 | Room context |
| Video Publishers | `0x0222` | Binary | Complete current publication list |
| Video Codec | `0x0223` | String | Room's active video codec |

The list is always complete, never a delta. A client MUST treat each notification as replacing its entire view of the room's video state.

A newly joined video-capable client receives a Video Status as part of learning the room; servers SHOULD send it immediately after the Join Voice Room (600) reply rather than adding fields to that reply, so that voice's reply shape is untouched.

Video Status is the **discovery** half of the extension and Video Subscribe (610) is the **selection** half. A client learns from 611 what exists, decides what it wants, and asks for it with 610. Nothing else announces a publication, and nothing else delivers one.

---

## Media Transport

### Track-to-User Mapping

The voice extension maps tracks to users through SDP `a=mid` attributes. This extension extends that grammar:

| `mid` value | Meaning | Defined by |
|---|---|---|
| `send` | The client's own microphone | Voice |
| `user-{UID}` | Audio from that user | Voice |
| `cam-send` | The client's own camera | This document |
| `cam-user-{UID}` | Camera video from that user | This document |
| `scr-send` | The client's own screen share | This document |
| `scr-user-{UID}` | Screen-share video from that user | This document |

`{UID}` is the decimal Hotline user ID, `1`–`65535`, with no leading zeros, exactly as in the voice extension. All the voice extension's rules about mids carry over unchanged: a mid is never reassigned to a different user or kind; a stopped or departed publication keeps its section with `a=inactive` and port 9; `m=` lines are never deleted, so `sdpMLineIndex` values remain stable; and a client MUST tolerate a mid it does not recognise, mirroring the section in its answer without mapping it to a user.

**New sections are appended, never inserted.** A room where video starts and stops repeatedly accumulates inactive sections, which is the cost of stable indices and is bounded by the participant count times the number of kinds.

**Mid values MUST NOT exceed 16 bytes.** This is not stylistic. Mainstream WebRTC stacks negotiate the MID RTP header extension (`urn:ietf:params:rtp-hdrext:sdes:mid`), and the one-byte header form of [RFC 8285](https://datatracker.ietf.org/doc/html/rfc8285) carries at most 16 bytes of extension data; a longer MID cannot be represented. The prefixes above are abbreviated for exactly this reason — `scr-user-65535` is 14 bytes, whereas the more readable `screen-user-65535` would be 17 and could not be carried. Implementations SHOULD reject a longer mid rather than truncate it.

### Annotated SDP Offer Example

A server's offer to user 5, in a room where user 12 is in voice with camera on and user 23 is in voice sharing a screen. User 5 has started a camera publication of its own **and has subscribed to both of the other publications** — an offer to a user 5 that had subscribed to neither would carry the three audio sections and `cam-send`, and no others. Annotations (`#` lines) are not part of the SDP; session-level and repeated per-section attributes (ICE credentials, fingerprint, `setup`) follow the voice extension and are elided after the first occurrence for brevity.

```
v=0
o=- 1234567890 2 IN IP4 0.0.0.0
s=-
t=0 0
a=group:BUNDLE user-12 user-23 send cam-user-12 scr-user-23 cam-send
a=msid-semantic: WMS

# Audio from user 12 — unchanged from the voice extension
m=audio 9 UDP/TLS/RTP/SAVPF 0
c=IN IP4 0.0.0.0
a=mid:user-12
a=rtpmap:0 PCMU/8000
a=sendonly
a=rtcp-mux
a=setup:actpass
a=ice-ufrag:srvr
a=ice-pwd:servericepasswordvalue1234
a=fingerprint:sha-256 AA:BB:...:99

# Audio from user 23
m=audio 9 UDP/TLS/RTP/SAVPF 0
a=mid:user-23
a=rtpmap:0 PCMU/8000
a=sendonly
...

# User 5's microphone
m=audio 9 UDP/TLS/RTP/SAVPF 0
a=mid:send
a=rtpmap:0 PCMU/8000
a=recvonly
...

# Camera video from user 12 (server sends, client receives)
m=video 9 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
b=AS:1500
a=mid:cam-user-12
a=rtpmap:96 VP8/90000
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtcp-fb:96 ccm fir
a=sendonly
a=rtcp-mux
a=setup:actpass
a=ice-ufrag:srvr
a=ice-pwd:servericepasswordvalue1234
a=fingerprint:sha-256 AA:BB:...:99
a=ssrc:1111111111 cname:video-12

# Screen share from user 23 — note a=content:slides
m=video 9 UDP/TLS/RTP/SAVPF 96 97
b=AS:2500
a=mid:scr-user-23
a=content:slides
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtcp-fb:96 ccm fir
a=sendonly
...
a=ssrc:2222222222 cname:screen-23

# User 5's own camera (client sends, server receives)
m=video 9 UDP/TLS/RTP/SAVPF 96 97
b=AS:1500
a=mid:cam-send
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtcp-fb:96 ccm fir
a=recvonly
...
```

Direction attributes are written from the offerer's — the server's — perspective, as in the voice extension: sections whose media the server forwards to the client are `a=sendonly`, and the client's own capture sections are `a=recvonly`. The client's answer mirrors each direction and replaces the ICE credentials and fingerprint with its own, with `a=setup:active`.

The server's offer for each video section MUST contain `a=rtpmap:96 VP8/90000`, the three `a=rtcp-fb` lines above, and — on a screen-share section — `a=content:slides`. It SHOULD contain a `b=AS` line reflecting the configured ceiling for that kind.

### Send SSRC Declaration

The voice extension requires the client's answer to declare the SSRC of its microphone stream, and permits a server to fall back to payload-type matching when it does not.

**For video sections there is no fallback.** The client's answer MUST declare `a=ssrc:<ssrc> cname:<cname>` in every video section it is sending on — `cam-send`, `scr-send`, or both — and the SSRC MUST match the SSRC of the RTP packets it actually sends.

The reason is [Payload Types](#payload-types): camera and screen video are both VP8 at payload type 96, arriving on one bundled transport within one peer connection. Payload type cannot distinguish them, and the server has no other means to decide whether an inbound video packet is a face or a spreadsheet. A server receiving an answer whose video send sections omit `a=ssrc` MUST reject the answer for those sections and MUST NOT guess: forwarding a screen share to the tiles where faces belong is worse than not forwarding it.

Standard WebRTC stacks emit `a=ssrc` automatically for every sending track. Implementations that hand-write SDP must take care.

### Renegotiation

Video renegotiation uses Voice SDP Offer (602) and Voice SDP Answer (603) and obeys the voice extension's serialisation rule without amendment: the server MUST NOT send a peer a new offer while a previous offer to that peer is unanswered; changes that accumulate meanwhile are applied internally and released as one consolidated offer when the answer arrives.

Video makes that rule matter more, because it multiplies the events that mutate a peer's offer: joining voice, leaving voice, a subscribed publisher stopping or disconnecting, and the peer's own subscription and publication changes. Implementations SHOULD treat "the offer describing this peer's own publications and subscriptions as they now stand" as a value computed on demand from current state, rather than maintaining a queue of deltas — the consolidation the rule requires then falls out for free, and a burst of five changes during one outstanding offer produces exactly one follow-up offer.

**Opt-in receive is what keeps this bounded.** Were every publication mirrored into every peer, one camera starting in a room of sixteen would renegotiate fifteen peer connections at once, and a room where people switch cameras on and off would spend its time in offer/answer. Because a publication reaches only its subscribers, starting one renegotiates the publisher alone; each other peer renegotiates at most once, when and if its user asks to see it.

Pause and resume deliberately do **not** renegotiate. Subscription changes do, which is why [Video Subscribe (610)](#video-subscribe-610) takes the client's whole desired set at once rather than one stream per request.

### Keyframes and RTCP Feedback

Video introduces a requirement audio does not have: a decoder cannot start mid-stream. A receiver that begins receiving a video stream sees nothing until the next keyframe, and VP8 encoders left alone may not produce one for many seconds.

**Servers MUST request a keyframe from a publisher when a new receiver begins consuming that publication.** Concretely, on each of:

- a subscription being added (Video Subscribe 610) — the common case, and the one that decides how fast a tile appears when a user clicks it;
- a receiver's renegotiation completing for a section that has become active;
- resume after a pause (Video State 609);
- a receiver's own RTCP PLI or FIR arriving for that stream.

The request is sent to the publisher as RTCP Picture Loss Indication ([RFC 4585](https://datatracker.ietf.org/doc/html/rfc4585)) or Full Intra Request ([RFC 5104](https://datatracker.ietf.org/doc/html/rfc5104)); the `a=rtcp-fb` lines in the offer are what make them legal.

**Servers MUST rate-limit keyframe requests per publication.** A room of eight receivers whose renegotiations complete together will otherwise ask one publisher for eight keyframes in a few milliseconds, and the publisher will send eight — a bitrate spike precisely when the network is busiest. A single request per publication per interval, with **one second RECOMMENDED**, coalesces the burst into the one keyframe that satisfies all of them.

Publishers MUST honour PLI and FIR by producing a keyframe promptly.

**Retransmission.** A receiver's NACK MUST NOT be discarded unanswered. The server either answers it from a retransmission cache of its own, sending the lost packet again — as an RTX packet (PT 97, `apt=96`) on a section where RTX was negotiated, otherwise as the original packet — or forwards it toward the publisher, whose own retransmission then reaches the loss. A server MAY do both, answering what its cache holds and forwarding the rest. RTX counts as negotiated only when the receiver's answer keeps PT 97: offering it is not enough, and a hand-rolled client may well drop it. Video loss is far more visible than audio loss; a stream with no loss recovery at all degrades badly on ordinary domestic connections.

The SFU forwards RTP payloads, sequence numbers and timestamps unchanged, and rewrites the SSRC as the voice extension does: each forwarded copy carries the SSRC declared for that section in the receiving peer's most recent offer. Feedback travels the other way. A receiver's PLI, FIR and NACK name the SSRC that receiver sees, and the server MUST translate them to the publisher's own SSRC before acting on them or passing them on.

### Bitrate

PCMU is 64 kbps and needs no management. Video is not, and this specification takes the deliberately simple position that **the server's ceilings are configuration, not negotiation**:

- The server advertises its per-kind ceilings in `DATA_VIDEO_LIMITS` at login and reflects them in `b=AS` on each video section.
- Clients MUST NOT exceed the advertised resolution, frame rate or bitrate for a kind. A client whose camera cannot be constrained to the ceiling MUST NOT publish.
- Servers MUST NOT trust clients to comply. A publication persistently exceeding its ceiling MAY be stopped by the server, with a Video Status (611) reflecting it.

Servers and clients MAY additionally negotiate `goog-remb` or `transport-cc` for genuine congestion control, and clients SHOULD honour REMB when the server sends it. Interoperability MUST NOT depend on either. Full congestion control is left to a future revision; the per-room publication caps are what bound the problem in this one.

### Session Failure

The voice extension's timeout table governs the peer connection as a whole and is unchanged. Video adds one rule:

**A failed video publication MUST NOT tear down the voice session.** If a publisher's video RTP stops arriving while its audio continues, the server SHOULD stop the publication — releasing its slot and sending Video Status (611) and Voice Room Status (605) — and leave audio untouched. The reverse is also true: a client whose camera fails locally sends Video Stop and remains in the call. Losing video is a degradation; losing the call is a failure.

The media-timeout in the voice extension's table applies to the peer connection: a peer sending neither audio nor video RTP nor RTCP for the timeout is gone, and is torn down as before.

---

## Room Model

Video rooms are voice rooms. There is no separate video room, no separate room ID space, and no separate join.

- A participant must be in voice in room `cid` before publishing video there.
- Leaving voice, being kicked from the chat, or disconnecting ends every publication in that room and releases its slots.
- The one-room-at-a-time rule applies transitively: joining voice in room B implicitly leaves room A, which ends every publication in A.
- The voice extension's [Room Membership](Capabilities-Voice.md#room-membership) rules govern who may be in the room at all, including the requirement that room identifiers be allocated by the server from a cryptographically secure random source. Video adds no new addressing and therefore no new exposure.

**Screen-share slots are room-wide.** With the default `VideoMaxScreensPerRoom` of 1, a second participant's Video Start for the screen kind is refused while another is sharing. The server MUST NOT preempt the existing share; the error text SHOULD say that someone else is sharing so the client can say so plainly. Operators who want a free-for-all raise the setting.

---

## Access Privileges

Two new privilege bits in the standard 64-bit access bitmap carried by `FieldUserAccess` (110):

| Bit | Name | Description |
|---|---|---|
| 59 | `AccessVideoChat` | User may publish camera video in voice rooms |
| 60 | `AccessScreenShare` | User may publish a screen share in voice rooms |

Bit 59 is the next available bit after `AccessMessaging` (bit 58); the preceding extension allocations are `AccessVoiceChat` (55), chat history (56), `AccessSendMedia` (57) and `AccessMessaging` (58).

**Screen share has its own bit deliberately.** Showing your face and showing your desktop are different trust decisions: a screen share can leak documents, credentials and other people's messages, in a way a camera generally cannot, and an operator may reasonably permit one and not the other. Servers SHOULD default `AccessScreenShare` off for guest accounts.

**Behaviour:**

- Video Start (607) for the camera kind is refused without bit 59; for the screen kind, without bit 60. Neither bit implies the other, and neither implies nor is implied by `AccessVoiceChat` (55) — although in practice a user without voice access can never reach a room to publish in.
- `CAPABILITY_VIDEO` is echoed in the login reply regardless of these bits, so clients show a disabled control with an explanatory tooltip rather than hiding it.
- Receiving video requires no privilege bit beyond being in the room.
- Servers that do not implement per-account privileges MAY treat both bits as always set.

---

## Bandwidth Considerations

VP8 at the default camera ceiling (1280×720, 30 fps, 1.5 Mbps) and screen ceiling (1920×1080, 15 fps, 2.5 Mbps). Unlike PCMU, video bitrate is variable; these are ceilings, and a still image costs far less than a moving one.

Camera-only room, worst case — every participant publishing at the ceiling **and** every participant subscribed to every other. Because subscription is opt-in, this is an upper bound rather than an expectation; a room where most participants watch only the speaker costs a small fraction of it.

| Participants | Upload per client | Download per client | Server total forwarding |
|---|---|---|---|
| 2 | 1.5 Mbps | 1.5 Mbps | 3 Mbps |
| 4 | 1.5 Mbps | 4.5 Mbps | 18 Mbps |
| 8 (default cap) | 1.5 Mbps | 10.5 Mbps | 84 Mbps |

Server forwarding grows as N×(N−1) — quadratically — which is why `VideoMaxCamerasPerRoom` defaults to 8 while the voice extension's `VoiceMaxPerRoom` defaults to 16. The room can hold sixteen people; eight of them can have cameras on.

Adding one screen share at its ceiling costs each other participant 2.5 Mbps of download and the server 2.5×(N−1) Mbps of forwarding.

Practical guidance for operators: a 360p/500 kbps camera ceiling is entirely adequate for a grid of small tiles and cuts these figures by a factor of three.

The download column is the one a client controls, and it controls it entirely: subscribing to two of eight cameras costs two streams, not eight. Clients on metered or mobile connections SHOULD subscribe narrowly — the visible tiles, or the active speaker — and revise the set as the view changes. The upload column is fixed by the ceiling and is the same whether one peer is watching or fifteen; the SFU sends N copies, the publisher sends one.

---

## Client Behaviour

- **Capability advertisement:** set bits 2 and 10 at login. If bit 10 is not echoed, disable all video UI and send no video transactions.
- **Limits:** read `DATA_VIDEO_LIMITS` from the login reply and configure encoders within the advertised ceilings before joining.
- **Publishing:** send Video Start (607), wait for the resulting Voice SDP Offer (602), answer it with `a=ssrc` present in the send section, and **begin capturing only once the negotiation completes**. As with voice, RTP sent before the session is established is discarded by the transport.
- **Camera default:** clients SHOULD start with the camera off and publish only on an explicit user action, for the same reason the voice extension says to join muted.
- **Pause, don't stop:** use Video State (609) for a camera toggle. Reserve Video Stop (608) for ending a share, so that screen-share slots are released promptly.
- **Receiving is a decision:** a client receives no video until it sends Video Subscribe (610). Learn what exists from Video Status (611), subscribe to what will actually be displayed, and revise the set when the view changes — a collapsed panel, a backgrounded window or a scrolled-away tile should not keep costing bandwidth.
- **Subscribe in one request:** send the complete desired set, not one stream per request. Four separate subscribes cost four renegotiations serialised behind one another; one subscribe naming four streams costs one.
- **Subscribe slightly ahead:** a stream takes a renegotiation and a keyframe to appear, so a client that subscribes only at the moment the user clicks will show a visible delay. Where bandwidth allows, subscribing to what is about to be shown is worth the traffic.
- **Rendering:** lay out streams by `mid` and by the kind reported in Video Status (611). Never key on `sdpMLineIndex` or section order.
- **Unknown mids:** a section whose mid this client does not recognise MUST be mirrored in the answer and MUST NOT be rendered as any user's video.
- **Paused streams:** show the participant's tile as present-but-paused rather than removing it; a paused publication still exists.
- **Screen share:** obtain explicit user consent for each share, using the platform's own picker where one exists, and display a persistent indicator while sharing. See [Privacy and Abuse](#privacy-and-abuse).
- **Keyframes:** send RTCP PLI when a decoder has no keyframe. Do not spin: one request per second per stream is ample.
- **Failure:** a video failure is not a call failure. Drop the publication, keep the call, and tell the user what happened.
- **No camera, no problem:** a client with no camera participates fully, receiving video and publishing none.
- **Device changes:** switching cameras, or changing what a screen share captures, is not a new publication. The client keeps the publication and its SSRC, swaps the source beneath the encoder (in WebRTC, `RTCRtpSender.replaceTrack`, not removing the track and adding another), and sends no transaction. It MUST NOT use Video Stop (608) and Video Start (607) for this: stopping releases the slot, which another participant may take, and announces a stop and a start to the room. It cannot declare a new SSRC instead, because only the server makes offers, and inbound video is attributed by the SSRC the last answer declared. RTP sequence numbers and timestamps MUST continue across the change, as the voice extension requires for audio: the server forwards them unchanged. The publisher SHOULD begin the new source with a keyframe, since a change of resolution cannot be decoded from the old stream's reference frames.
- **Voice-only clients:** a client that negotiated voice but not video SHOULD still show who is publishing, from the [publication bits](#publication-bits) of the voice participant list.

---

## Server Behaviour

- **Per-client offers:** build each peer's offer from the room's current state and that peer's negotiated capabilities. Never send video sections to a client that has not confirmed `CAPABILITY_VIDEO`, and never forward video RTP to one.
- **Slot enforcement:** enforce `VideoMaxCamerasPerRoom` and `VideoMaxScreensPerRoom` at Video Start, and release slots on stop, leave, kick and disconnect.
- **Pause enforcement:** discard a paused publication's inbound RTP rather than forwarding it, exactly as for a muted audio stream.
- **Publication bits:** set bits 1 and 2 of each voice participant entry from that participant's publications, in the join reply and in every room status, and send a room status when a publication starts or stops. See [Publication Bits](#publication-bits).
- **Subscribe enforcement:** deliver a publication only to peers subscribed to it. Retain subscriptions naming publications that do not exist yet, and activate them if they appear. Never tell a publisher who is subscribed to it.
- **Keyframes:** request one whenever a receiver newly begins consuming a publication, and rate-limit to roughly one per second per publication.
- **No transcoding:** never decode or re-encode. Payloads, sequence numbers and timestamps are forwarded unchanged; only the SSRC is rewritten, to the one each receiver's offer declared. The server never sees pixels.
- **Codec enforcement:** offer only VP8 at PT 96. Reject an answer whose video sections lack PT 96, or whose send sections lack `a=ssrc`.
- **Renegotiation:** obey the voice extension's per-peer serialisation rule; consolidate accumulated changes into one follow-up offer.
- **Cleanup:** ending a participant's voice session ends its publications. A publication whose media has stopped may be reaped without disturbing the call.
- **Logging:** log publication start and stop, with user, room and kind. **Never log, record, decode or store media content.** A screen share is by nature the most sensitive stream a Hotline server will ever carry.
- **Metrics:** active publications by kind, publications per room, and forwarded bitrate.

---

## Privacy and Abuse

Screen sharing raises risks that voice does not, and a specification that ignores them invites every implementation to solve them differently.

- **Consent is per share, never persistent.** A client MUST obtain explicit user consent each time a screen share begins, and MUST NOT offer a "remember this choice" affordance for it. Where the platform provides a system picker with its own consent step, clients SHOULD use it rather than enumerating windows themselves.
- **The indicator is not optional.** While sharing, a client MUST display a persistent, visible indication of what is being shared. Screen shares that the user has forgotten about are the extension's characteristic failure mode.
- **Consent to a room, not to a server.** An implicit leave (joining voice in another room) ends every publication; it MUST NOT carry a screen share into the new room. A user who wants to share in room B says so in room B.
- **No silent starts.** A server MUST NOT be able to start a client's camera or screen share. There is no server-initiated publication in this specification, and none should be added.
- **A new source is a new choice.** Changing what a screen share captures takes the same consent as starting one, and the sharing indicator MUST update to show the new source.
- **Others are in the frame.** Servers SHOULD make the room's publication state visible to every participant, video-capable or not: through Video Status (611) to video-capable participants, and through the [publication bits](#publication-bits) of Voice Room Status (605) and the Join Voice Room (600) reply to all of them, so that a voice-only participant can at least know a camera is on. Clients SHOULD show who is publishing regardless of whether they render the video.
- **Kick ends everything.** A participant removed from a chat loses voice and every publication in that room immediately, per the voice extension's existing rule.

---

## Hotline-ng Binding

> This section is specific to the Hotline-ng protocol — hxd-ng's JSON/WebSocket frontend, specified in [hotline-ng.md](hotline-ng.md) — and is not part of the classic Hotline extension above. It exists so that one room serves both wires. Implementers of the classic extension can stop reading here.

The media plane is shared: an ng client and a classic client in the same room exchange video through the same SFU, over the same UDP port, negotiated with the same SDP. Only the signalling encoding differs, and the mapping is mechanical. As on the classic wire, video reuses the voice binding's `voice_offer` / `voice_answer` / `voice_ice` for all SDP and ICE traffic — those messages carry video sections without changing shape.

The login reply's `caps` list gains `"video"` when the SFU supports video, alongside `"voice"`. As on the classic wire, `"video"` never appears without `"voice"`.

**Requests:**

| `req` | params | ok | errors |
|---|---|---|---|
| `video_start` | `cid`, `kind` (`"camera"` \| `"screen"`) | `{ "codec": "VP8" }` | `video_disabled`, `access_denied`, `not_in_voice`, `already_publishing`, `video_full`, `rate_limited` |
| `video_stop` | `cid`, `kind?` (omit for all) | `{}` | `not_in_voice` |
| `video_state` | `cid`, `kind`, `paused` (bool) | `{}` | `not_in_voice`, `not_publishing` |
| `video_subscribe` | `cid`, `streams` (array of `{ uid, kind }`; `[]` for none) | `{}` | `not_in_voice`, `rate_limited` |

**Events:**

| `ev` | data |
|---|---|
| `video_status` | `{ cid, publishers: [ { uid, kind, paused } ] }` |

`rate_limited` carries `retry_after`, in whole seconds, as it does everywhere on this wire: a server may bound how often one session starts video or changes its subscriptions, since each renegotiates others. Stop and pause are never refused this way.

`kind` is a string on this wire rather than the classic wire's integer, `paused` is a boolean rather than a flags word, and `streams` is an array of objects rather than a packed blob — for the same reason the voice binding sends a participant array: the transport is already JSON and a mobile client should not be decoding bit fields. `codec` is a string on both wires and needs no translation.

`video_subscribe` carries the complete desired set on this wire too, and the same defaults apply: a session receives no video until it asks, and `"streams": []` turns it all off in one request. For a mobile client this is the whole point of the binding — the ng protocol exists for clients on cellular connections, and this is the message that keeps a video room affordable on one.

`video_status` carries the complete publication list on every emission, exactly as transaction 611 does. A client replaces its whole view of the room's video state on each one.

The [publication bits](#publication-bits) have no counterpart here: `voice_status` keeps its `{ uid, muted }` entries, because a session on this wire in a room where video exists has `"video"` in its `caps` and receives `video_status` whatever it renders. `voice_status` is still emitted when a publication starts or stops, as 605 is, carrying an unchanged list.

**Limits** are reported in the login reply rather than as a repeated field:

```jsonc
"caps": ["voice", "video"],
"video": {
  "camera": { "max_width": 1280, "max_height": 720, "max_fps": 30,
              "max_bitrate": 1500000, "max_per_room": 8 },
  "screen": { "max_width": 1920, "max_height": 1080, "max_fps": 15,
              "max_bitrate": 2500000, "max_per_room": 1 }
}
```

**Detach and resume.** Video follows voice: publications end when the session's connection is lost, because the peer connection dies with it. A resumed session finds itself out of voice and therefore out of video, learns so from the replayed `voice_status` and `video_status` in its outbox tail, and republishes explicitly if the user wants it. A phone that dropped its WebSocket dropped its UDP path too; a fresh ICE and DTLS handshake is faster and more honest than pretending otherwise.

**Browser clients** get this nearly free. `RTCPeerConnection` supplies VP8, `a=ssrc`, NACK/PLI and RTX without configuration, and `getDisplayMedia` supplies the screen share with the browser's own consent UI and sharing indicator — which satisfies the [Privacy and Abuse](#privacy-and-abuse) requirements by construction rather than by discipline.

---

## Implementation Notes

- **WebRTC libraries.** Any RFC-compliant stack can implement this. [str0m](https://github.com/algesten/str0m) (Rust, sans-I/O) supports VP8, RTX, PLI/FIR feedback and per-`mid` streams through the same direct API the voice SFU already uses; [Pion](https://github.com/pion/webrtc) (Go) and libwebrtc are the other obvious choices. Browsers need nothing beyond `RTCPeerConnection` and `getDisplayMedia`.
- **The mid parser is the first thing to change** in any existing voice implementation. A parser that recognises exactly `send` and `user-{UID}` must grow the four new prefixes, keep its strictness about leading zeros and uid 0, and enforce the 16-byte ceiling. It is a small change and it is load-bearing: mid parsing is the whole track-to-user mapping.
- **A voice SFU that keys receive state by `mid` alone will collide** once a participant publishes both a camera and a screen, because both arrive on one bundled transport within one peer connection. Key inbound video by SSRC — the SSRC the answer declared — and use the mid only for labelling. (Implementations that already key receive bins by transport-level pad rather than by mid have solved this in advance.)
- **The offer builder** gains a per-section kind and a wider attribute set: `rtpmap` for VP8 and RTX, three `rtcp-fb` lines, `b=AS`, and `a=content:slides` on screen sections. The template stays a template.
- **The answer parser** gains a per-section `a=ssrc` requirement for video and must reject rather than fall back.
- **Keyframe request plumbing is genuinely new work** — there is no analogue in an audio-only SFU. Budget for the rate limiter as part of it, not as a later optimisation; without it the first eight-person room reveals the problem immediately.
- **The predictable support question** is a client that publishes happily and sees nothing from anyone, because its author expected streams to arrive unbidden. Implementations SHOULD log a subscription set that stays empty while publications exist, and client authors should reach for Video Subscribe (610) first when video does not appear.
- **For hxd-ng specifically, on abuse:** inbound video is policed per publication at `[voice] police_factor` times its kind's ceiling, a publication still far over after ten seconds is stopped with the Video Status the bitrate section allows for, video starts and subscription changes that add a stream someone is publishing are drawn from a per-session allowance, and pause flips share voice's debounce. See [voice.md](voice.md) §13.
- **For hxd-ng specifically:** the `VoiceMedia` trait grows publication methods (`publish`, `unpublish`, `set_paused`) plus `set_subscriptions`, and its `MediaEvent` grows a keyframe-request and a publication-failed variant; the domain gains publication and slot state, and a per-peer subscription set, next to the existing per-room participant state, where the existing four cleanup paths already reach. The subscription set is per (uid, room) and is the natural place for the "retain a subscription to a publication that does not exist yet" rule, since the domain is what learns about publications appearing. The offer template, the answer parser and the forwarding loop are the `hxd-voice` half. Configuration extends the existing `[voice]` section rather than adding a port, since the transport is shared.
- **For GtkHx specifically:** rendering is the large piece — a decode path and a video widget where none exists — and the GStreamer pipeline gains `vp8enc`/`vp8dec` alongside the existing `mulawenc`/`mulawdec` legs. `hxproto`'s `MidLabel` and its participants parser are shared with hxd-ng and change once, in the shared crate, for both.

---

## Reserved for Future Revisions

Named here so that the allocations are held and the omissions are visibly deliberate:

- **Screen audio** — sharing a window's sound alongside its picture. Kind ID 3 and the mid prefixes `sca-send` / `sca-user-{UID}` are reserved for it. It is a second audio stream per participant, which the current audio path does not expect.
- **Simulcast** — a publisher sending several qualities and the server choosing per receiver. Video Subscribe (610) lets a receiver choose *whether* to receive a stream; simulcast would let it choose *at what quality*, which is the finer-grained answer to heterogeneous networks. The natural shape is a quality field alongside each entry in `DATA_VIDEO_SUBSCRIPTIONS`, which is why that structure is a packed array of fixed-size entries rather than a bare list of user IDs. It also needs RID and `a=simulcast`. Transaction IDs 612–619 and field IDs `0x0226`–`0x023F` are reserved partly for it.
- **Additional codecs** — H.264 for hardware decode on constrained clients, AV1 for bitrate. Codec IDs 1–3 are reserved. Any addition must confront the no-transcoding constraint: a room agrees on one codec, so a mixed-codec room needs either a negotiated room codec at join time or per-publication codecs with receiver-side capability filtering.
- **Congestion control** — `transport-cc` with a real bandwidth estimator, replacing fixed ceilings.
- **Recording** — deliberately not specified. It is a policy question before it is a protocol one, and a room where recording is possible but unsignalled is worse than one where it is impossible.
