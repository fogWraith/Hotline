# Client–Tracker Communication

**Specification for implementing tracker listing retrieval in a Hotline client**

---

## Table of Contents

- [Overview](#overview)
- [Transport and Addressing](#transport-and-addressing)
- [Connection Flow](#connection-flow)
- [Handshake](#handshake)
  - [Client Request](#client-request)
  - [Tracker Response](#tracker-response)
  - [Handshake Validation](#handshake-validation)
- [Receiving the Server List](#receiving-the-server-list)
  - [Batch Header](#batch-header)
  - [Server Record Format](#server-record-format)
  - [Reading Multiple Batches](#reading-multiple-batches)
  - [Batch Reading Algorithm (Python Example)](#batch-reading-algorithm)
- [Complete Protocol Sequence](#complete-protocol-sequence)
- [String Decoding](#string-decoding)
- [Error Handling](#error-handling)
- [HTTP Tunneling](#http-tunneling)
- [Protocol Version 2 (Unused)](#protocol-version-2-unused)
- [Implementation Example — List Retrieval](#implementation-example--list-retrieval)

---

## Overview

A Hotline client connects to a tracker server to retrieve the list of currently registered Hotline servers. The interaction is a brief TCP exchange:

1. Client connects to the tracker's TCP port.
2. Client sends a 6-byte magic handshake.
3. Tracker responds with a 6-byte magic acknowledgment.
4. Tracker sends the server list as one or more batches.
5. Connection closes.

The entire exchange is read-only from the client's perspective — the client sends only the 6-byte handshake and then reads until the connection closes.

---

## Transport and Addressing

| Parameter       | Value                    |
|-----------------|--------------------------|
| Transport       | TCP                      |
| Destination port| 5498 (default)           |
| Direction       | Client → Tracker (handshake), Tracker → Client (server list) |

The client connects to the tracker's **TCP** port (default 5498). This is distinct from the server registration port (UDP/5499).

---

## Connection Flow

```
Client                              Tracker
  │                                    │
  │──── TCP connect ──────────────────►│
  │                                    │
  │──── Magic (6 bytes) ─────────────►│
  │     "HTRK" + version 0x0001       │
  │                                    │
  │◄─── Magic (6 bytes) ──────────────│
  │     "HTRK" + version 0x0001       │
  │                                    │
  │◄─── Batch header (8 bytes) ───────│
  │◄─── Server records ──────────────│
  │     ... (variable count)           │
  │                                    │
  │◄─── Batch header (8 bytes) ───────│  ← only if more batches
  │◄─── Server records ──────────────│
  │     ...                            │
  │                                    │
  │◄─── Connection close ────────────│
  │                                    │
```

---

## Handshake

### Client Request

The client sends exactly 6 bytes immediately after the TCP connection is established:

| Offset | Size | Field        | Type  | Value        | Description               |
|--------|------|--------------|-------|--------------|---------------------------|
| 0      | 4    | Magic string | bytes | `"HTRK"`     | `0x48 0x54 0x52 0x4B`     |
| 4      | 2    | Version      | u16   | `0x0001`     | Protocol version 1 (big-endian) |

**Hex representation:** `48 54 52 4B 00 01`

### Tracker Response

The tracker responds with an identical 6-byte magic:

| Offset | Size | Field        | Type  | Value        | Description               |
|--------|------|--------------|-------|--------------|---------------------------|
| 0      | 4    | Magic string | bytes | `"HTRK"`     | `0x48 0x54 0x52 0x4B`     |
| 4      | 2    | Version      | u16   | `0x0001`     | Protocol version 1 (big-endian) |

### Handshake Validation

After receiving the tracker's 6-byte response, the client should validate:

1. **Magic string matches** `"HTRK"` (`0x4854524B`). If not, close the connection — this is not a Hotline tracker.
2. **Version is 1** (`0x0001`). If the version is unexpected, the client may still proceed (for forward compatibility) or close the connection.

---

## Receiving the Server List

After the handshake, the tracker immediately sends the server list. The list is organized into one or more **batches**, each preceded by an 8-byte header.

### Batch Header

Each batch begins with an 8-byte header:

| Offset | Size | Field          | Type | Description                                     |
|--------|------|----------------|------|-------------------------------------------------|
| 0      | 2    | Message type   | u16  | Always `0x0001` (server list message, big-endian)|
| 2      | 2    | Data size      | u16  | Size in bytes of remaining data in this batch (big-endian) |
| 4      | 2    | Total servers  | u16  | Total server count across **all** batches (big-endian) |
| 6      | 2    | Batch count    | u16  | Number of server records in **this** batch (big-endian) |

**Key relationships:**
- `Total servers` is the same value in every batch header for a given response.
- `Batch count` may differ between batches (the last batch may be smaller).
- `Data size` covers the server records in this batch only (not the header itself).

### Server Record Format

Each server record immediately follows the batch header (or the previous server record) within a batch:

| Offset | Size | Field            | Type   | Description                              |
|--------|------|------------------|--------|------------------------------------------|
| 0      | 4    | IP address       | u32    | IPv4 address (4 octets, network order)   |
| 4      | 2    | Port             | u16    | Server's TCP listening port (big-endian) |
| 6      | 2    | User count       | u16    | Connected user count (big-endian)        |
| 8      | 2    | Reserved         | u16    | Reserved field, typically `0x0000`       |
| 10     | 1    | Name length      | u8     | Length of server name in bytes           |
| 11     | var  | Name             | bytes  | Server name string                       |
| N      | 1    | Description len  | u8     | Length of description in bytes           |
| N+1    | var  | Description      | bytes  | Server description string                |

**Fixed portion:** 11 bytes (IP + port + users + reserved + name length).
**Minimum record size:** 12 bytes (11 fixed bytes + 1 byte for description length, with empty name and description).

**IP address decoding:** The 4 bytes represent individual octets in network order. For example, bytes `0xC0 0xA8 0x01 0x64` decode to IP address `192.168.1.100`.

### Reading Multiple Batches

The server list may span multiple batches. This occurs when the total serialized data is too large for a single batch (typically when it would exceed ~8 KiB).

A client must continue reading batch headers and server records until it has accumulated a number of servers equal to the `Total servers` value from the first batch header.

### Batch Reading Algorithm (Python Example)

```python
total_servers = 0
servers = []
first_batch = true

while len(servers) < total_servers or first_batch:
    header = read_bytes(8)
    msg_type, data_size, total_count, batch_count = unpack(header)

    if first_batch:
        total_servers = total_count
        first_batch = false

    for i in range(batch_count):
        record = read_server_record()
        servers.append(record)

close_connection()
return servers
```

**Edge cases:**
- `Total servers = 0`: The tracker has no registered servers. The batch header will have `Batch count = 0` and no server records follow.
- `Batch count = 0` in a subsequent header with `Total servers > 0`: This should not occur with well-behaved trackers, but implementations should handle it gracefully.
- Connection closed mid-transfer: Return whatever servers were successfully read. The partial list may still be useful to the user.

---

## Complete Protocol Sequence

### Sequence Diagram

```
Client                              Tracker
  │                                    │
  │  1. TCP SYN/ACK                    │
  │──────────────────────────────────►│
  │                                    │
  │  2. Send HTRK magic (6 bytes)     │
  │  48 54 52 4B 00 01                │
  │──────────────────────────────────►│
  │                                    │
  │  3. Receive HTRK magic (6 bytes)  │
  │  48 54 52 4B 00 01                │
  │◄──────────────────────────────────│
  │                                    │
  │  4. Receive batch header (8 bytes)│
  │  00 01 XX XX 00 6A 00 61          │  ← type=1, size=XX, total=106, batch=97
  │◄──────────────────────────────────│
  │                                    │
  │  5. Receive 97 server records     │
  │  [IP][Port][Users][Rsvd][Name]... │
  │◄──────────────────────────────────│
  │                                    │
  │  6. Receive batch header (8 bytes)│
  │  00 01 XX XX 00 6A 00 09          │  ← type=1, size=XX, total=106, batch=9
  │◄──────────────────────────────────│
  │                                    │
  │  7. Receive 9 server records      │
  │◄──────────────────────────────────│
  │                                    │
  │  8. Connection closed by tracker  │
  │◄──────────────────────────────────│
  │                                    │
  │  Total: 97 + 9 = 106 servers ✓   │
```

### Wire Capture Example — Single Server

A tracker with one registered server named "My Server" (description: "Welcome!") at `192.168.1.100:5500` with 3 users:

```
Client → Tracker (6 bytes):
  48 54 52 4B 00 01                     HTRK + version 1

Tracker → Client (6 bytes):
  48 54 52 4B 00 01                     HTRK + version 1

Tracker → Client (8 + 30 bytes):
  00 01                                 Message type = 1
  00 1E                                 Data size = 30
  00 01                                 Total servers = 1
  00 01                                 Batch count = 1

  C0 A8 01 64                           IP = 192.168.1.100
  15 7C                                 Port = 5500
  00 03                                 Users = 3
  00 00                                 Reserved
  09                                    Name length = 9
  4D 79 20 53 65 72 76 65 72            "My Server"
  08                                    Desc length = 8
  57 65 6C 63 6F 6D 65 21              "Welcome!"
```

---

## String Decoding

Server name and description strings are encoded as Pascal-style strings:

```
[1 byte: length N] [N bytes: string data]
```

### Character Encoding

- **Classic Hotline clients** use **MacRoman** encoding.
- **Modern servers** may send **UTF-8** encoded strings.
- **Obsession** (Qt 4) decodes with Shift-JIS support for Japanese servers, using a `TextHelper::DecodeText()` utility.
- **PhareRouge** (Java) defaults to ISO-8859-1 but supports configurable encoding for internationalization.
- Clients could / should attempt UTF-8 decoding first, falling back to MacRoman (or Latin-1 as an approximation) if UTF-8 decoding fails.

---

## Error Handling

### Connection Failures

| Error                    | Handling                                          |
|--------------------------|---------------------------------------------------|
| Connection refused       | Tracker is down or wrong port. Report to user.    |
| Connection timeout       | Tracker unreachable. Apply timeout (10–30 sec).   |
| Magic mismatch           | Not a Hotline tracker. Close and report.          |
| Unexpected version       | Log warning, attempt to continue reading.         |
| Truncated batch header   | Connection lost mid-transfer. Return partial list.|
| Truncated server record  | Discard incomplete record. Return partial list.   |

### Timeouts

Clients should apply timeouts at two stages:

1. **Connection timeout:** 10–30 seconds for TCP connect + handshake.
2. **Read timeout:** 30–60 seconds for the complete server list transfer. Large trackers with hundreds of servers may take several seconds.

### Empty Tracker

A tracker with no registered servers will send a valid batch header with `Total servers = 0` and `Batch count = 0`. No server records follow. This is a normal condition, not an error.

---

## SOCKS5 Proxy Support

SilverWing (BeOS) implements full SOCKS5 proxy support for tracker connections, including username/password authentication. This allows clients behind corporate firewalls to browse tracker listings through a SOCKS5 proxy server.

The proxy handshake occurs before the HTRK magic exchange:
1. Connect to SOCKS5 proxy
2. Negotiate authentication method (none or username/password)
3. Send CONNECT request to tracker address
4. On success, proceed with normal HTRK handshake over the proxied connection

---

## Invalid IP Filtering

Some client implementations filter out invalid server records before displaying them:

- **SilverWing:** Skips server records where the first IP octet is `0x00` (e.g., `0.x.x.x`), consuming the remaining bytes of the record to stay in sync but not adding it to the list.

---

## Multi-Batch Compatibility Notes

Not all client implementations handle multi-batch responses correctly:

| Client           | Multi-batch | Notes                                           |
|------------------|-------------|------------------------------------------------|
| Hotline (macOS)  | Yes         | Full batch header parsing                       |
| Hotline (Tauri)  | Yes         | Reads until total count reached                |
| Mobius           | Yes         | Well-tested                                    |
| PhareRouge       | Yes         | Uses `pack` field for page tracking            |
| Obsession        | Partial     | Reads `numberOfServers` from first header only |
| SilverWing       | **No**      | Reads 14-byte header, iterates `nservers` records — no continuation |

Tracker implementations should prefer smaller batch sizes to maximize compatibility with clients that don't handle multi-batch responses.

---

## HTTP Tunneling

When a client is behind an HTTP proxy that blocks raw TCP connections, the Hotline protocol supports HTTP tunneling for tracker access:

- The client establishes an HTTP connection to the tracker.
- The client does **not** send the HTRK magic handshake.
- The tracker detects the HTTP transport and responds with the server list in the standard format (batch headers + server records) wrapped in HTTP response body.

**Note:** HTTP tunneling for tracker access is rarely used in practice and not all tracker implementations support it.

---

## Protocol Version 2 (Unused)

The protocol specification mentions version 2, which would add login/password fields to the client handshake:

| Description   | Size | Data  | Note                          |
|---------------|------|-------|-------------------------------|
| Login size    | 1    | ≥ 31  | Login string size             |
| Login         | size |       | Login string (padded with 0)  |
| Password size | 1    | ≥ 31  | Password string size          |
| Password      | size |       | Password string (padded with 0) |

**Version 2 was never implemented.** All known tracker implementations reject version 2 clients by immediately closing the connection. Clients should always send version 1 (`0x0001`).

---

## Implementation Example — List Retrieval

### Python (asyncio)

```python
import asyncio
import socket
import struct

TRACKER_MAGIC = b"HTRK"
TRACKER_VERSION = 1

async def fetch_tracker_list(host: str, port: int = 5498, timeout: float = 30.0):
    """Connect to a Hotline tracker and retrieve the server list."""

    reader, writer = await asyncio.wait_for(
        asyncio.open_connection(host, port),
        timeout=timeout,
    )

    try:
        # Send handshake
        magic = TRACKER_MAGIC + struct.pack("!H", TRACKER_VERSION)
        writer.write(magic)
        await writer.drain()

        # Read handshake response
        response = await asyncio.wait_for(reader.readexactly(6), timeout=timeout)
        resp_magic, resp_version = response[:4], struct.unpack("!H", response[4:6])[0]

        if resp_magic != TRACKER_MAGIC:
            raise ValueError(f"Invalid tracker magic: {resp_magic!r}")

        # Read server list
        servers = []
        total_servers = None

        while total_servers is None or len(servers) < total_servers:
            # Read batch header (8 bytes)
            header = await asyncio.wait_for(reader.readexactly(8), timeout=timeout)
            msg_type, data_size, total_count, batch_count = struct.unpack("!4H", header)

            if total_servers is None:
                total_servers = total_count

            if total_servers == 0:
                break

            # Read server records in this batch
            for _ in range(batch_count):
                # Fixed fields: IP(4) + port(2) + users(2) + reserved(2) = 10 bytes
                fixed = await reader.readexactly(10)
                ip_bytes = fixed[0:4]
                port_num = struct.unpack("!H", fixed[4:6])[0]
                users = struct.unpack("!H", fixed[6:8])[0]
                # fixed[8:10] is reserved

                ip_str = socket.inet_ntoa(ip_bytes)

                # Name (Pascal string)
                name_len = (await reader.readexactly(1))[0]
                name = (await reader.readexactly(name_len)).decode("utf-8", "replace")

                # Description (Pascal string)
                desc_len = (await reader.readexactly(1))[0]
                desc = (await reader.readexactly(desc_len)).decode("utf-8", "replace")

                servers.append({
                    "ip": ip_str,
                    "port": port_num,
                    "users": users,
                    "name": name,
                    "description": desc,
                })

        return servers

    finally:
        writer.close()
        await writer.wait_closed()

def print_servers(servers):
    """Print server list in a formatted table."""
    if not servers:
        print("No servers found.")
        return

    print(f"\n{'Server Name':<30} {'IP':<15} {'Port':<6} {'Users':<6} {'Description'}")
    print("-" * 90)
    
    for server in servers:
        name = server['name'][:28] + ".." if len(server['name']) > 30 else server['name']
        desc = server['description'][:30] + ".." if len(server['description']) > 32 else server['description']
        print(f"{name:<30} {server['ip']:<15} {server['port']:<6} {server['users']:<6} {desc}")
    
    print(f"\nTotal: {len(servers)} server(s)")

async def main():
    host = "hltracker.com"
    port = 5498
    
    print(f"Connecting to {host}:{port}...")
    
    try:
        servers = await fetch_tracker_list(host, port)
        print(f"Successfully retrieved {len(servers)} servers")
        print_servers(servers)
        
    except asyncio.TimeoutError:
        print("Error: Connection timed out", file=sys.stderr)
        sys.exit(1)
    except ConnectionRefusedError:
        print(f"Error: Could not connect to {host}:{port}", file=sys.stderr)
        sys.exit(1)
    except Exception as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    asyncio.run(main())
```

