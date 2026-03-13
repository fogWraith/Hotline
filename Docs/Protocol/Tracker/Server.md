# Server–Tracker Communication

**Specification for implementing tracker registration in a Hotline server**

---

## Table of Contents

- [Overview](#overview)
- [Transport and Addressing](#transport-and-addressing)
- [Registration Datagram](#registration-datagram)
  - [Fixed Header](#fixed-header)
  - [Variable Data](#variable-data)
  - [Complete Datagram Layout](#complete-datagram-layout)
- [PassID — Server Instance Identifier](#passid--server-instance-identifier)
- [Registration Lifecycle](#registration-lifecycle)
  - [Startup](#startup)
  - [Heartbeat Loop](#heartbeat-loop)
  - [Shutdown](#shutdown)
- [Configuration](#configuration)
  - [Tracker Server List](#tracker-server-list)
  - [Registration Interval](#registration-interval)
  - [Advertised Port](#advertised-port)
  - [Source Address Binding](#source-address-binding)
- [Password Authentication](#password-authentication)
- [String Encoding](#string-encoding)
- [Field Size Limits](#field-size-limits)
- [Multiple Trackers](#multiple-trackers)

---

## Overview

A Hotline server registers its presence with one or more tracker servers by periodically sending UDP datagrams. These datagrams contain the server's name, description, listening port, current user count, and a random instance identifier. The tracker stores this information and makes it available to clients that query for active servers.

Registration is a **fire-and-forget** operation — the server sends a UDP datagram and receives no acknowledgment. The server must continue sending heartbeats at a regular interval to maintain its listing. If heartbeats stop, the tracker will expire the entry (typically after 10–31 minutes depending on the tracker implementation).

---

## Transport and Addressing

| Parameter       | Value                    |
|-----------------|--------------------------|
| Transport       | UDP                      |
| Destination port| 5499 (default)           |
| Direction       | Server → Tracker (unidirectional) |
| Acknowledgment  | None (fire-and-forget)   |

The server sends each heartbeat as a single UDP datagram. There is no connection state, no response from the tracker, and no retransmission mechanism.

**Important:** The tracker's **UDP registration port (5499)** is distinct from its **TCP listing port (5498)**. Server registration always uses UDP/5499.

---

## Registration Datagram

### Fixed Header

The first 12 bytes of every registration datagram form a fixed-size header:

| Offset | Size | Field       | Type | Description                                      |
|--------|------|-------------|------|--------------------------------------------------|
| 0      | 2    | Version     | u16  | Protocol version, always `0x0001` (big-endian)   |
| 2      | 2    | Port        | u16  | Server's TCP listening port (big-endian)          |
| 4      | 2    | User count  | u16  | Current number of connected users (big-endian)    |
| 6      | 2    | Reserved    | u16  | Set to `0x0000`                                   |
| 8      | 4    | PassID      | u32  | Random server instance identifier (big-endian)    |

### Variable Data

Immediately following the 12-byte header are three Pascal-style strings (1-byte length prefix followed by string data):

| Field           | Encoding     | Description                                   |
|-----------------|--------------|-----------------------------------------------|
| Name            | 1-byte len + data | Server name (displayed in tracker listings) |
| Description     | 1-byte len + data | Server description                         |
| Password        | 1-byte len + data | Tracker authentication password (empty if none required) |

### Complete Datagram Layout

```
Byte:  0    1    2    3    4    5    6    7    8    9   10   11   12  ...
     ├────────┼────────┼────────┼────────┼─────────────────┼────┼───...
     │Version │  Port  │ Users  │Reserved│     PassID      │NLen│ Name
     │ 0x0001 │        │        │ 0x0000 │                 │    │
     └────────┴────────┴────────┴────────┴─────────────────┴────┴───...
                                                             │
     ...──┼────┼────────────...──┼────┼────────────...──┤    │
          │DLen│ Description     │PLen│ Password         │    │
          │    │                 │    │                   │    │
     ...──┴────┴────────────...──┴────┴────────────...──┘    │
                                                              └─ Pascal strings
```

**Minimum datagram size:** 15 bytes (12-byte header + 3 zero-length Pascal strings: `0x00 0x00 0x00`).

---

## PassID — Server Instance Identifier

The PassID is a random 32-bit unsigned integer generated once when the server process starts. It remains constant for the entire lifetime of the server process.

**Purpose:**
- Allows the tracker to recognize the same logical server even if its IP address or port changes (e.g., behind NAT, after a network configuration change).
- Enables the tracker to distinguish between a server restart (new PassID) and a heartbeat (same PassID).

**Generation:** Use a cryptographically secure random number generator. The PassID must not be zero.

**Lifetime:** Generate once at server startup. Do not change the PassID during the server's runtime. A new PassID is generated on each server restart.

**Known fixed PassIDs:** Some implementations use hardcoded rather than random values. PhareRouge (Java) uses the constant `0xefe44d00`. Tools like mhxd's `tspoof.c` document known fingerprint values:

| PassID         | Software                  |
|----------------|---------------------------|
| `0xefe44d00`   | PhareRouge / early betas  |
| `0xa67985b1`   | Hotline Server v1.2.3     |
| `0x486d1f7c`   | AfterBirth a2c3           |
| `0x7f23a207`   | Hotline Server v1.8.4     |

Using a fixed PassID is not recommended — it prevents the tracker from distinguishing between instances.

---

## Registration Lifecycle

### Startup

1. Generate a random 32-bit PassID.
2. Determine the advertised port (from configuration or the actual listening port).
3. Send an initial registration datagram to each configured tracker.
4. Start the heartbeat timer.

### Heartbeat Loop

The server sends a registration datagram to all configured trackers at a fixed interval.

**Standard / default interval:** 300 seconds (5 minutes).

Most implementations use 300 seconds. Notable exceptions:

| Implementation   | Interval    | Notes                                      |
|------------------|-------------|--------------------------------------------|
| GLoarbLine       | 300s (5 min)| Standard                                   |
| synhxd/shxd      | ~310s       | Slightly longer than 5 min                 |
| hlserver         | 300s (5 min)| Standard                                   |
| SilverWing       | 180s (3 min)| Shorter interval (BeOS)                    |
| PhareRouge       | 300s (5 min)| Standard (Java)                            |

Each heartbeat updates the tracker with the server's current state (user count, name, description). The tracker refreshes the entry's timestamp, preventing expiry.

### Shutdown

No explicit deregistration is required. When the server stops sending heartbeats, the tracker will expire the entry after its configured timeout (typically 10–31 minutes).

There is no "deregister" packet in the Hotline tracker protocol. The server simply stops sending heartbeats.

---

## Configuration

### Tracker Server List

A server may register with multiple trackers simultaneously. Each tracker entry requires:

| Field    | Type            | Required | Description                           |
|----------|-----------------|----------|---------------------------------------|
| address  | string (hostname or IP) | Yes | Tracker hostname or IP address  |
| port     | integer         | Yes      | Tracker UDP port (default: 5499)      |
| password | string or null  | No       | Tracker authentication password       |

**HLServer Example configuration:**
```json
{
  "tracker": {
    "enabled": true,
    "servers": [
      {
        "address": "hltracker.com",
        "port": 5499,
        "password": null
      },
      {
        "address": "tracker2.example.com",
        "port": 5499,
        "password": "secretpass"
      }
    ],
    "interval": 300.0
  }
}
```

### Registration Interval

| Parameter | Default | Minimum | Description                                  |
|-----------|---------|---------|----------------------------------------------|
| interval  | 300.0   | 5.0     | Seconds between heartbeats to each tracker   |

The 300-second (5-minute) interval is used by all known implementations (GLoarbLine, synhxd, phxd, Mobius) and is the de facto standard. One should not reduce this significantly, as it adds unnecessary load to tracker servers.

### Advertised Port

By default, the server advertises the port it is actually listening on. Some deployments require advertising a different port (e.g., port forwarding, reverse proxy). An `advertise_port` configuration option allows overriding the port sent in the registration datagram without changing the actual listening port.

### Source Address Binding

For servers with multiple network interfaces, a `source_address` option allows binding the outgoing UDP socket to a specific local address. This is useful when the server needs to register from a specific interface.

---

## Password Authentication

If a tracker requires a password for registration, the password is included in the registration datagram as the third Pascal string.

**Protocol behavior:**
- The password is sent in **cleartext** over UDP. There is no encryption or hashing.
- If the tracker does not require a password, the password field should be an empty Pascal string (a single `0x00` byte for zero length).
- If the password is incorrect, the tracker silently discards the datagram. No error response is sent.

**Security note:** The tracker password provides minimal security — it prevents casual spam but is trivially intercepted on an unencrypted network. It is not a substitute for network-level security.

---

## String Encoding

All strings in the registration datagram use Pascal-style encoding:

```
[1 byte: length] [N bytes: string data]
```

- **Length prefix:** A single unsigned byte (`u8`) containing the string length in bytes.
- **Maximum length:** 255 bytes (limited by the 1-byte length prefix).
- **Encoding:** Historically MacRoman. Modern implementations should send UTF-8. The tracker will store and relay whatever bytes are received.
- **No null terminator.** The string is exactly `length` bytes long.

### Encoding Conversion

Some implementations convert strings to legacy encodings before sending to ensure compatibility with classic Hotline clients:

- **SilverWing** (BeOS): Converts server name/description from UTF-8 to a configurable legacy encoding (Shift-JIS, ISO-8859-1, etc.) before packing into the datagram.
- **PhareRouge** (Java): Uses `String.getBytes(encoding)` with a default of ISO-8859-1 but supports configurable encoding for Japanese (Shift-JIS) locales.
- **HLServer**: Sends UTF-8 natively.

---

## Field Size Limits

| Field       | Max Length | Notes                                          |
|-------------|------------|------------------------------------------------|
| Name        | 255 bytes  | Truncate if longer; some trackers limit to 31  |
| Description | 255 bytes  | Truncate if longer; some trackers limit to 31  |
| Password    | 255 bytes  | Empty string (length 0) if no password         |
| Port        | 0–65535    | Standard TCP port range                        |
| User count  | 0–65535    | Capped by u16                                  |
| PassID      | 0x00000001–0xFFFFFFFF | Should not be zero                  |

**Note:** Some older tracker implementations (synhxd, GLoarbLine) limit the name field to 31 bytes. Modern implementations should accept up to 255 bytes but truncation to 31 is safe for maximum compatibility.

---

## Multiple Trackers

When registering with multiple trackers:

1. Send the heartbeat to each tracker independently.
2. Each tracker may have a different password. Use the password configured for that specific tracker entry.
3. A failure to reach one tracker should not prevent sending to others.
4. The same PassID must be used for all trackers (it identifies the server instance, not the tracker relationship).

