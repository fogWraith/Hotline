# Hotline Tracker Server

**Specification for implementing a Hotline Connect Tracker**

---

## Table of Contents

- [Overview](#overview)
- [Network Architecture](#network-architecture)
- [Ports and Listeners](#ports-and-listeners)
- [Protocol Constants](#protocol-constants)
- [TCP Listener — Client List Requests](#tcp-listener--client-list-requests)
  - [Handshake](#handshake)
  - [Server List Response](#server-list-response)
  - [Batch Header](#batch-header)
  - [Server Record Format](#server-record-format)
  - [Batching Logic](#batching-logic)
- [UDP Listener — Server Registration](#udp-listener--server-registration)
  - [Registration Datagram Format](#registration-datagram-format)
  - [Registration Processing](#registration-processing)
  - [Duplicate Detection](#duplicate-detection)
  - [Password Verification](#password-verification)
- [Server Expiry](#server-expiry)
- [Banning and Access Control](#banning-and-access-control)
- [Protocol Version 2 (Abandoned)](#protocol-version-2-abandoned)
- [Extended Protocols — TRXL and TRXR](#extended-protocols--trxl-and-trxr)
- [Implementation Notes](#implementation-notes)

---

## Overview

The Hotline Tracker is a directory service that maintains a list of active Hotline servers. It serves two roles:

1. **Accepts registrations** from Hotline servers via UDP datagrams (port 5499).
2. **Serves server listings** to Hotline clients via TCP connections (port 5498).

Servers periodically send UDP heartbeat packets to register or refresh their presence. The tracker maintains these entries and expires them if a server stops sending heartbeats. Clients connect via TCP, perform a brief handshake, and receive the complete server list.

---

## Network Architecture

```
┌──────────────┐     UDP/5499      ┌──────────────────┐     TCP/5498      ┌──────────────┐
│              │ ─────────────────► │                  │ ◄─────────────── │              │
│  HL Server   │   Registration    │  Tracker Server   │   List Request   │  HL Client   │
│              │   (heartbeat)     │                  │ ─────────────────►│              │
└──────────────┘                   └──────────────────┘   Server List     └──────────────┘
```

- Servers send **UDP datagrams** to the tracker at a fixed interval (typically every 300 seconds / 5 minutes).
- Clients open a **TCP connection**, perform a handshake, receive the full server list, and disconnect.
- The tracker never initiates connections to servers or clients.

---

## Ports and Listeners

| Transport | Port | Direction             | Purpose                          |
|-----------|------|-----------------------|----------------------------------|
| TCP       | 5498 | Client → Tracker      | Server list requests             |
| UDP       | 5499 | Server → Tracker      | Registration heartbeats          |

Both ports should be configurable. The defaults are defined by the Hotline protocol.

---

## Protocol Constants

| Constant         | Value            | Description                          |
|------------------|------------------|--------------------------------------|
| `HTRK_MAGIC`     | `"HTRK\x00\x01"` | 6-byte magic for v1 protocol         |
| `HTRK_MAGIC_LEN` | 6                | Length of magic bytes                |
| `HTRK_TCPPORT`   | 5498             | Default TCP listener port            |
| `HTRK_UDPPORT`   | 5499             | Default UDP listener port            |
| `SIZEOF_HTRK_HDR`| 12               | Minimum registration datagram size   |

**Note:** All numeric data transmitted over the wire is in network byte order (big-endian).

---

## TCP Listener — Client List Requests

### Handshake

When a client connects to the tracker on TCP port 5498, the following handshake occurs:

**Client sends** (6 bytes):

| Description  | Size | Data         | Note                                      |
|--------------|------|--------------|-------------------------------------------|
| Magic string | 4    | `"HTRK"`     | `0x4854524B`                               |
| Version      | 2    | `0x0001`     | Protocol version 1 (big-endian)            |

**Tracker responds** (6 bytes):

| Description  | Size | Data         | Note                                      |
|--------------|------|--------------|-------------------------------------------|
| Magic string | 4    | `"HTRK"`     | `0x4854524B`                               |
| Version      | 2    | `0x0001`     | Echo back protocol version 1              |

Upon receiving anything other than a valid v1 handshake, the tracker should close the connection. After responding with the magic, the tracker immediately sends the server list and then closes the connection.

### Server List Response

The server list is sent as one or more **batches**. Each batch begins with an 8-byte header followed by a sequence of variable-length server records.

### Batch Header

Each batch starts with the following 8-byte header:

| Offset | Size | Field           | Type | Description                                |
|--------|------|-----------------|----- |--------------------------------------------|
| 0      | 2    | Message type    | u16  | Always `0x0001` (server list message)      |
| 2      | 2    | Data size       | u16  | Size in bytes of remaining data in batch   |
| 4      | 2    | Total servers   | u16  | Total server count across **all** batches  |
| 6      | 2    | Batch count     | u16  | Number of server records in **this** batch |

The `Total servers` field must be identical across all batch headers for a given response. The `Batch count` contains the number of server records that follow this header.

### Server Record Format

Each server record immediately follows the batch header (or the previous server record) and has the following structure:

| Offset | Size | Field            | Type   | Description                              |
|--------|------|------------------|--------|------------------------------------------|
| 0      | 4    | IP address       | u32    | IPv4 address (4 octets, network order)   |
| 4      | 2    | Port             | u16    | Server's TCP listening port (big-endian) |
| 6      | 2    | User count       | u16    | Connected user count (big-endian)        |
| 8      | 2    | Flags            | u16    | Reserved, typically `0x0000`             |
| 10     | 1    | Name length      | u8     | Length of server name (max 255)          |
| 11     | var  | Name             | bytes  | Server name string                       |
| N      | 1    | Description len  | u8     | Length of description (max 255)          |
| N+1    | var  | Description      | bytes  | Server description string                |

**String encoding:** Historically MacRoman (classic Hotline). Modern implementations may use UTF-8. Strings are length-prefixed (1-byte length, Pascal-style), not null-terminated.

**IP address:** Stored as 4 individual octets in network byte order. To represent `192.168.1.100`, the bytes are `0xC0 0xA8 0x01 0x64`.

### Batching Logic

Server records must be split across multiple batches when the total serialized data would exceed the batch size limit. The batch size threshold varies by implementation:

| Implementation   | Batch Size Limit  | Notes                                    |
|------------------|-------------------|------------------------------------------|
| synhxd/shxtrackd/mhxd/shxd | 8191 bytes (`0x1FFF`) | Standard C implementations    |
| hltracker        | 30,000 bytes      | Much larger batches (VB6)                |
| GLoarbLine       | ~8 KiB            | Similar to synhxd                        |

Algorithm:

1. Initialize a buffer with an 8-byte batch header.
2. For each server in the list:
   a. Serialize the server record.
   b. If adding this record would exceed 8191 bytes, finalize the current batch (fill in `Data size` and `Batch count`), write it to the TCP connection, and start a new batch.
   c. Otherwise, append the record to the current buffer and increment the batch count.
3. After the last server, finalize and send the final batch.
4. Close the TCP connection.

**Example:** A tracker with 106 servers might produce:
- Batch 1: header (`Total servers=106`, `Batch count=97`) + 97 server records
- Batch 2: header (`Total servers=106`, `Batch count=9`) + 9 server records

Clients must continue reading batch headers until they have accumulated `Total servers` entries.

---

## UDP Listener — Server Registration

### Registration Datagram Format

Servers register by sending a single UDP datagram to the tracker on port 5499. The datagram has the following structure:

**Fixed header** (12 bytes):

| Offset | Size | Field       | Type | Description                                      |
|--------|------|-------------|------|--------------------------------------------------|
| 0      | 2    | Version     | u16  | Protocol version, always `0x0001` (big-endian)   |
| 2      | 2    | Port        | u16  | Server's TCP listening port (big-endian)          |
| 4      | 2    | User count  | u16  | Number of connected users (big-endian)            |
| 6      | 2    | Reserved    | u16  | Reserved, set to `0x0000`                         |
| 8      | 4    | PassID      | u32  | Random server instance identifier (big-endian)    |

**Variable data** (immediately follows the header):

| Offset | Size | Field           | Type  | Description                                   |
|--------|------|-----------------|-------|-----------------------------------------------|
| 12     | 1    | Name length     | u8    | Length of server name (max 255)               |
| 13     | var  | Name            | bytes | Server name string                             |
| N      | 1    | Description len | u8    | Length of description (max 255)               |
| N+1    | var  | Description     | bytes | Server description string                      |
| M      | 1    | Password len    | u8    | Length of tracker password (0 if none)         |
| M+1    | var  | Password        | bytes | Tracker authentication password                |

**Minimum datagram size:** 12 bytes (header only, with empty name/description/password).

### Registration Processing

When a UDP datagram arrives, the tracker should:

1. **Validate size:** Discard datagrams smaller than 12 bytes (`SIZEOF_HTRK_HDR`).
2. **Parse header:** Extract version, port, user count, and PassID using big-endian byte order.
3. **Verify version:** Accept only version `0x0001`. Discard unknown versions.
4. **Parse strings:** Extract name, description, and password as Pascal-style strings (1-byte length prefix followed by data).
5. **Check ban list:** If the source IP is banned, discard silently.
6. **Verify password:** If the tracker requires authentication, compare the received password against the configured password list. Reject if no match.
7. **Check server limit:** If a maximum server count is configured, reject new registrations that would exceed it.
8. **Detect duplicates:** Check if this server already exists (see [Duplicate Detection](#duplicate-detection)).
9. **Insert or update:** Add a new entry or update the existing one with the current timestamp, user count, name, and description.

### Duplicate Detection

The tracker must detect when a server is re-registering (heartbeat) versus a new server appearing. Multiple strategies are used, checked in order:

| Priority | Match Criteria        | Action                                          |
|----------|-----------------------|-------------------------------------------------|
| 1        | Same IP + same port   | Update existing entry (same server, same socket)|
| 2        | Same PassID           | Update existing entry (server restarted on different port or IP changed) |
| 3        | No match              | Create new entry                                |

The **PassID** is a random 32-bit number generated by the server at startup. It remains constant for the lifetime of the server process, allowing the tracker to recognize the same server even if its IP or port changes (e.g., behind NAT or after a port change).

**Implementation notes:**
- GLoarbLine also checks by name CRC, allowing same-named server updates to coalesce.
- synhxd uses a 256-bucket hash table indexed by IP address octets for O(1) average lookup.
- Simpler implementations can use a dictionary keyed by `(IP, port)` with a secondary lookup by PassID.

### Password Verification

If the tracker is configured with one or more passwords, the password field in the registration datagram must match one of them. If the password does not match:

- Discard the datagram silently (do not send any response — this is UDP).
- Optionally log the rejection.

If the tracker has no password configured, accept all registrations regardless of the password field.

---

## Server Expiry

The tracker must periodically remove entries for servers that have stopped sending heartbeats. There are two common approaches:

### Timestamp-Based Expiry (Recommended)

Each server entry stores the timestamp of its last successful registration. A periodic cleanup task compares the current time against each entry's timestamp.

**Recommended expiry threshold:** 600 seconds (10 minutes), which allows for 2 missed heartbeats at the standard 300-second interval.

**Reference values from existing implementations:**
- GLoarbLine: 1860 seconds (31 minutes) — very conservative
- phxd: 300 seconds (5 minutes) — aggressive, only 1 missed heartbeat allowed
- A reasonable default is 600–900 seconds (10–15 minutes)

### Counter-Based Expiry

synhxd/shxtrackd/mhxd/shxd use a counter approach: each entry has a `clock` field that increments on every timer tick. When a server re-registers, its clock resets to 0. When the clock reaches 2 (two full intervals without re-registration), the entry is removed.

With a typical timer interval of 900 seconds (shxtrackd default) or 360 seconds (mhxd/shxd default), this means servers survive 2 × interval without re-registering (30 minutes or 12 minutes respectively).

### Expiry Comparison

| Implementation     | Method     | Interval / Threshold  | Effective Timeout       |
|--------------------|------------|----------------------|-------------------------|
| GLoarbLine         | Timestamp  | 1860 seconds         | 31 minutes              |
| phxd               | Timestamp  | 300 seconds          | 5 minutes               |
| hltracker          | Timestamp  | 15 minutes (default) | 15 minutes (configurable) |
| synhxd/shxtrackd   | Counter    | 900s interval, clock≥2 | ~30 minutes            |
| mhxd/shxd          | Counter    | 360s interval, clock≥2 | ~12 minutes            |

### Cleanup Timer Interval

The cleanup timer should run more frequently than the expected registration interval. Common values:

- Registration interval: 300 seconds (5 minutes)
- Cleanup timer: 60–300 seconds (1–5 minutes)

---

## Banning and Access Control

### IP Ban List

The tracker may maintain a ban list of IP addresses or patterns. Banned addresses are rejected for both:

- **UDP registrations** — datagrams from banned IPs are silently discarded.
- **TCP list requests** — connections from banned IPs are refused or immediately closed.

Some implementations maintain **separate ban lists** for registrations and list requests (shxtrackd, mhxd, shxd). This allows banning a server from registering while still allowing its users to browse, or vice versa.

### Ban Format

synhxd/shxtrackd/mhxd support wildcard patterns using `*` to match any octet:

```
192.168.1.*      # Bans all addresses in 192.168.1.0/24
10.0.*.*         # Bans all addresses in 10.0.0.0/16
203.0.113.42     # Bans a specific address
```

Default registration bans typically include `192.168.*` and `127.0.0.*` (private/loopback addresses).

### Content-Based Filtering

hltracker implements a rule-based filter system with five filter types applied per server record:

| Filter Type  | Matches On     | Options                              |
|-------------|----------------|--------------------------------------|
| Name        | Server name    | Exact or contains, case-sensitive    |
| Description | Description    | Exact or contains, case-sensitive    |
| Port        | Port number    | Min/max range                        |
| IP          | IP address     | Min/max IP range                     |
| User count  | User count     | Min/max range                        |

Filters have a priority order and a configurable default action (block or allow). Each rule can individually block or whitelist matching servers.

### Private IP Rejection

Some trackers automatically reject registration from private/reserved IP ranges (loopback, RFC 1918, multicast):
- `0.x.x.x`, `127.x.x.x`, `10.x.x.x`, `224.x.x.x+` — blocked
- `192.168.x.x`, `172.16-31.x.x` — blocked

### Per-IP Server Quota

GLoarbLine implements a configurable maximum number of servers per IP address. This prevents a single operator from flooding the tracker with entries. When the limit is reached, additional registrations from that IP are rejected.

---

## Protocol Version 2 (Abandoned)

The protocol defines version 2 (`0x0002`) in the handshake, but it was **never implemented** by any known tracker or client. All known implementations either:

- **Reject v2 clients** by closing the TCP connection immediately (phxd behavior).
- **Ignore v2 registrations** with empty handler stubs (synhxd behavior: `htrkx_udp_rcv()` is wrapped in `#if 0`).

Version 2 was intended to add login/password authentication to the client-tracker TCP handshake:

| Description   | Size | Data  | Note                          |
|---------------|------|-------|-------------------------------|
| Login size    | 1    | ≥ 31  | Login string size             |
| Login         | size |       | Login string (padded with 0)  |
| Password size | 1    | ≥ 31  | Password string size          |
| Password      | size |       | Password string (padded with 0) |

**Recommendation:** Implementations should send and accept only version 1. If a version 2 handshake is received, close the connection.

---

## Extended Protocols — TRXL and TRXR

synhxd defines two additional magic sequences in its TCP state machine:

| Magic          | Constant     | Intended Purpose                |
|----------------|--------------|---------------------------------|
| `"TRXL\x00\x01"` | `TRXL_MAGIC` | Extended list request protocol  |
| `"TRXR\x00\x01"` | `TRXR_MAGIC` | Extended registration protocol  |

These are recognized in the TCP connection state machine but **have no implementation**. The handler stubs are empty or disabled. No known client or server sends these sequences.

---

## Implementation Notes

As a small bonus, some additional information / food for thought.

### Data Structures

A tracker needs to efficiently support:
- **Lookup by (IP, port)** — for duplicate detection on UDP registration.
- **Lookup by PassID** — for server identity tracking across IP/port changes.
- **Iteration** — for generating the TCP server list response.
- **Expiry** — for periodic removal of stale entries.

A simple approach is a dictionary keyed by `(IP, port)` with a secondary index by PassID. For high-traffic trackers, a hash table indexed by IP (as used by synhxd) provides better performance.

### Concurrency

The TCP and UDP listeners operate independently:
- UDP registration can modify the server list while a TCP response is being generated.
- Use appropriate synchronization (mutex, lock, or copy-on-write) to prevent data races.

### String Handling

- Classic Hotline uses MacRoman encoding for strings.
- Modern trackers should accept UTF-8 gracefully, since modern servers may send UTF-8 encoded names.
- Name and description fields are limited to 255 bytes (1-byte length prefix).

### Custom Server Lists

Some trackers support loading a static list of "pinned" servers from a configuration file. These entries are prepended to the dynamic server list and do not expire. This is useful for featuring specific servers.

- **shxtrackd/synhxd:** `custom_list` file with format `ip:port,name,users,description` (supports DNS hostnames).
- **hltracker:** "Fake servers" added via GUI, stored in Windows registry. Fake entries bypass all filter rules and never expire.

### Fake Server Injection and Block Messages

hltracker supports two novel features using synthetic server records:

- **Fake servers:** The operator can inject permanent server entries (name, description, IP, port, user count) into the listing. These appear alongside real servers but never expire and bypass filters.
- **Block message as fake listing:** When a banned IP requests a listing, instead of refusing the connection, the tracker sends a single fake server record (IP `127.0.0.1`, port `5500`, user count `31337`) with the ban message as the description.

### Tracker Mirroring

hltracker can pull server lists from other tracker servers and merge them into its own listing. Mirrored servers are tagged as `Mirrored` type (distinct from `Real` and `Fake`), expire normally, but don't trigger new-server alerts. Mirroring is timer-driven with a configurable interval (default: 20 minutes).

### Software Version Query (HTVR)

hltracker defines a secondary TCP magic `"HTVR"` (4 bytes). When received, the tracker responds with an ASCII version string (e.g., `"Codebox Hotline Tracker v1.0.0 by rob"`) instead of a server listing. This is unique to this implementation and not part of the Hotline standard.

