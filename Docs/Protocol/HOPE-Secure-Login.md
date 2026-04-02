# HOPE – Hotline One-time Password Extension

HOPE adds secure password authentication and optional transport encryption to the Hotline login sequence. Instead of sending credentials using the standard bitwise-inversion scheme, HOPE allows the client and server to negotiate a MAC algorithm and use a one-time session key to compute a cryptographic hash of the password and optionally the login name.

## Table of Contents

- [Overview](#overview)
- [Transactions and Fields](#transactions-and-fields)
  - [HOPE Object IDs (Secure Login)](#hope-object-ids-secure-login)
  - [HOPE Object IDs (Transport Encryption)](#hope-object-ids-transport-encryption)
  - [Standard Hotline Fields Reused by HOPE](#standard-hotline-fields-reused-by-hope)
- [Negotiation Flow](#negotiation-flow)
  - [Step 1 — Client Identification](#step-1--client-identification)
  - [Step 2 — Server Identification Reply](#step-2--server-identification-reply)
  - [Step 3 — Authenticated Login](#step-3--authenticated-login)
- [MAC Algorithms](#mac-algorithms)
- [Session Key](#session-key)
- [Transport Encryption](#transport-encryption)
  - [Supported Ciphers](#supported-ciphers)
  - [Transport Key Derivation](#transport-key-derivation)
  - [Wire Format](#wire-format)
  - [Key Rotation](#key-rotation)
  - [Compression](#compression)
- [Compatibility](#compatibility)
- [App ID and App String Conventions](#app-id-and-app-string-conventions)

---

## Overview

HOPE is a security extension that piggybacks on the existing Login (107) transaction. No new transaction types are defined — the protocol reuses `htlcHdrLogin` (0x6B) for both the identification exchange and the final authenticated login.

A HOPE-capable client signals itself by sending a login packet with `DATA_LOGIN` set to a single null byte (`0x00`) and including a `DATA_HOPE_MAC_ALGORITHM` field. The server recognises this as a HOPE identification request and responds with a session key and algorithm selection instead of treating it as a real login attempt.

HOPE is also the **only** mechanism in the Hotline protocol for a client to advertise its application name and version to the server. Legacy clients are identified solely by the 16-bit version number in the handshake magic and the Version field (160) in the login transaction.

---

## Transactions and Fields

HOPE defines no new transaction types. All negotiation occurs within:

| Transaction | Hex ID | Decimal | Direction | Description |
|---|---|---|---|---|
| `htlcHdrLogin` | `0x0000006B` | 107 | Client → Server | Login (used for both identification and authentication) |
| `htlsHdrTask` | `0x00010000` | 65536 | Server → Client | Task reply (used for the identification reply) |

### HOPE Object IDs (Secure Login)

| ID (hex) | ID (dec) | Constant | Description |
|---|---|---|---|
| `0x0E01` | 3585 | `htlxDataAppID` | 4-byte application identifier (OSType or custom) |
| `0x0E02` | 3586 | `htlxDataAppString` | Human-readable app name and version string |
| `0x0E03` | 3587 | `htlxDataSessionKey` | Server-generated per-connection session key |
| `0x0E04` | 3588 | `htlxDataMACAlgorithm` | MAC algorithm list (client) or selection (server) |

### HOPE Object IDs (Transport Encryption)

These fields extend the original HOPE draft with cipher negotiation and transport encryption, as implemented by HotSocket/GLoarbline-family clients and HLServer.

| ID (hex) | ID (dec) | Constant | Description |
|---|---|---|---|
| `0x0EC1` | 3777 | `DATA_HOPE_SERVER_CIPHER` | Cipher(s) for server → client encryption |
| `0x0EC2` | 3778 | `DATA_HOPE_CLIENT_CIPHER` | Cipher(s) for client → server encryption |
| `0x0EC3` | 3779 | `DATA_HOPE_SERVER_CIPHER_MODE` | Cipher mode for server (e.g. `"STREAM"`) |
| `0x0EC4` | 3780 | `DATA_HOPE_CLIENT_CIPHER_MODE` | Cipher mode for client (e.g. `"STREAM"`) |
| `0x0EC5` | 3781 | `DATA_HOPE_SERVER_IV` | Initialisation vector for server cipher |
| `0x0EC6` | 3782 | `DATA_HOPE_CLIENT_IV` | Initialisation vector for client cipher |
| `0x0EC7` | 3783 | `DATA_HOPE_SERVER_CHECKSUM` | Checksum algorithm for server outbound |
| `0x0EC8` | 3784 | `DATA_HOPE_CLIENT_CHECKSUM` | Checksum algorithm for client outbound |
| `0x0EC9` | 3785 | `DATA_HOPE_SERVER_COMPRESSION` | Compression algorithm for server outbound |
| `0x0ECA` | 3786 | `DATA_HOPE_CLIENT_COMPRESSION` | Compression algorithm for client outbound |

### Standard Hotline Fields Reused by HOPE

| ID (hex) | ID (dec) | Field | Usage in HOPE |
|---|---|---|---|
| `0x0069` | 105 | User login | `0x00` in identification; MAC'd or inverted in authenticated login |
| `0x006A` | 106 | User password | MAC'd or inverted in authenticated login |
| `0x0066` | 102 | User name | Nickname (authenticated login only) |
| `0x0068` | 104 | User icon ID | Icon number (authenticated login only) |

---

## Negotiation Flow

### Step 1 — Client Identification

The client sends a `htlcHdrLogin` (107) packet with the following fields:

| ID | Field | Value | Notes |
|---|---|---|---|
| 105 | User login | `0x00` | Single null byte — signals HOPE identification |
| 106 | User password | `0x00` | Single null byte — placeholder |
| 0x0E04 | MAC Algorithm | `<u16:count> [<u8:len> <str:name>]+` | Ordered list of supported algorithms, most preferred first |
| 0x0E01 | App ID | `<str4:id>` | 4-byte application identifier |
| 0x0E02 | App String | `<str:desc>` | Human-readable name and version |
| 0x0EC2 | Client Cipher | Algorithm list (same encoding) | *Optional.* Ciphers the client supports for client → server |
| 0x0ECA | Client Compression | Algorithm list | *Optional.* Compression algorithms supported |
| 0x0E03 | Session Key | Empty (0 bytes) | Requests a session key from the server |

**MAC algorithm list encoding:**

```
<u16:count> [<u8:length> <str:name>]+
```

Example: three algorithms, `HMAC-SHA256` (preferred), `HMAC-SHA1`, and `HMAC-MD5`:
```
00 03  0B 48 4D 41 43 2D 53 48 41 32 35 36  09 48 4D 41 43 2D 53 48 41 31  08 48 4D 41 43 2D 4D 44 35
count  len "HMAC-SHA256"                     len "HMAC-SHA1"                 len "HMAC-MD5"
```

### Step 2 — Server Identification Reply

The server responds with a `htlsHdrTask` reply. The reply **must** have its `Type` field set to zero — classic clients (hx, gtkhx, shxd-family) read the first 4 bytes of the transaction header as a single big-endian uint32 and only recognise `0x00010000`. Setting `Type` to the login transaction type (`0x006B`) would produce `0x0001006B`, which these clients reject as an unknown header type.

| ID | Field | Value | Notes |
|---|---|---|---|
| 0x0E03 | Session Key | 64 bytes | Server-generated unique session key |
| 0x0E04 | MAC Algorithm | `0x0001 <u8:len> <str:name>` | Single chosen algorithm |
| 105 | User login | Empty or algorithm name | Empty = login is not MAC'd; algorithm name = login should be MAC'd |
| 0x0E01 | App ID | `<str4:id>` | Server's application identifier |
| 0x0E02 | App String | `<str:desc>` | Server's name and version |
| 0x0EC1 | Server Cipher | Algorithm list | *Optional.* Selected cipher for server → client |
| 0x0EC2 | Client Cipher | Algorithm list | *Optional.* Selected cipher for client → server |
| 0x0EC3 | Server Cipher Mode | String | *Optional.* e.g. `"STREAM"` |
| 0x0EC4 | Client Cipher Mode | String | *Optional.* e.g. `"STREAM"` |
| 0x0EC5 | Server IV | Bytes | *Optional.* Currently empty |
| 0x0EC6 | Client IV | Bytes | *Optional.* Currently empty |
| 0x0EC7 | Server Checksum | Bytes | *Optional.* Currently `"NONE"` |
| 0x0EC8 | Client Checksum | Bytes | *Optional.* Currently `"NONE"` |
| 0x0EC9 | Server Compression | Algorithm list | *Optional.* Selected compression |
| 0x0ECA | Client Compression | Algorithm list | *Optional.* Selected compression |

### Step 3 — Authenticated Login

The client sends a second `htlcHdrLogin` (107) with real credentials:

| ID | Field | Value | Notes |
|---|---|---|---|
| 105 | User login | `inverse(login)` or `mac(login, session_key)` | Inverted if server's login field was blank; MAC'd if non-blank |
| 106 | User password | `mac(password, session_key)` | Computed using the negotiated algorithm |
| 102 | User name | Display nickname | |
| 104 | User icon ID | Icon number | |
| 0x0EC1 | Server Cipher | Algorithm list | *Optional.* Echoes the confirmed server → client cipher from Step 2 |
| 0x0EC9 | Server Compression | Algorithm list | *Optional.* Echoes the confirmed compression from Step 2 |

> **Note:** Step 3 echoes the **server** cipher/compression selections (0x0EC1, 0x0EC9) back to the server — not the client fields (0x0EC2, 0x0ECA). This confirms the client accepted the negotiated parameters.

From this point, the normal Hotline login flow continues (agreement, user list, etc.). If transport encryption was negotiated, all subsequent packets are encrypted.

---

## MAC Algorithms

The following algorithms are defined, listed from strongest to weakest. Both client and server are **required** to support `INVERSE`.

| Algorithm | Computation | Output Size | Notes |
|---|---|---|---|
| `HMAC-SHA256` | `hmac_sha256(key=password, msg=session_key)` | 32 bytes | Preferred; ideal for AEAD key derivation |
| `HMAC-SHA1` | `hmac_sha1(key=password, msg=session_key)` | 20 bytes | |
| `SHA1` | `sha1(password + session_key)` | 20 bytes | |
| `HMAC-MD5` | `hmac_md5(key=password, msg=session_key)` | 16 bytes | |
| `MD5` | `md5(password + session_key)` | 16 bytes | |
| `INVERSE` | Bitwise NOT of each byte | Variable | Required fallback |

The server selects the strongest algorithm from the client's preference list that it also supports. If no match is found, the server falls back to `INVERSE`.

---

## Session Key

The session key is generated by the server for each connection. It consists of two parts:

| Component | Size | Description |
|---|---|---|
| Server IP | 4 bytes | IPv4 address of the server (big-endian, network byte order) |
| Server port | 2 bytes | Port number (big-endian) |
| Random bytes | 58 bytes | Cryptographically random |

**Total length:** 64 bytes.

The IP and port are taken from `getsockname()` on the accepted TCP socket — i.e., the server's local address on that specific connection (equivalent to Go's `conn.LocalAddr()`). This is the address the client connected **to**, not any configured or external address.

> **Security check:** Clients **validate** the embedded IP:port against the server address they used to connect. If the values do not match (e.g., due to NAT or a MITM proxy), clients will disconnect or warn the user. The shxd-family `hx` client prints `"<key_ip>:<key_port> != <connected_ip>:<connected_port>"` and closes the connection (overridable with `-f`).

---

## Transport Encryption

Transport encryption activates after a successful HOPE login when both sides negotiated a cipher. All further Hotline packets (header and body) are encrypted at the byte-stream level.

> **AEAD mode:** When `CHACHA20-POLY1305` is negotiated, the transport uses framed AEAD encryption instead of the stream cipher wire format described below. Key derivation, frame structure, nonce construction and file transfer encryption for AEAD mode are fully specified in [HOPE ChaCha20-Poly1305 Extension](HOPE-ChaCha20-Poly1305.md). The remainder of this section describes the stream cipher mode used by RC4 and Blowfish.

### Supported Ciphers

| ID | Name | Mode | Notes |
|---|---|---|---|
| 1 | `RC4` / `RC4-128` | Stream | ARC4 stream cipher |
| 2 | `BLOWFISH` | OFB | Output Feedback mode |
| 3 | `CHACHA20-POLY1305` | AEAD | ChaCha20-Poly1305 (RFC 8439); see [extension spec](HOPE-ChaCha20-Poly1305.md) |

Cipher name variants (`RC4`, `RC4-128`, `ARCFOUR`) are normalised to `RC4`. `CHACHA20` and `CHACHA20POLY1305` are normalised to `CHACHA20-POLY1305`. `NONE` disables encryption.

### Transport Key Derivation

After the password MAC is verified, transport keys are derived:

```
encode_key = MAC(key=password_bytes, msg=password_mac)
decode_key = MAC(key=password_bytes, msg=encode_key)
```

Where:
- `MAC` is the negotiated algorithm (must be MD5, SHA1, HMAC-MD5, HMAC-SHA1, or HMAC-SHA256 — not INVERSE)
- `password_bytes` is the plaintext password as UTF-8
- `password_mac` is the MAC computed during authentication: `mac(password, session_key)`

The server uses `encode_key` for outbound and `decode_key` for inbound. The client uses them in reverse.

### Wire Format

Once transport encryption is active, every Hotline packet is processed as follows:

**Outbound:**

1. Split packet into 20-byte header and variable-length body
2. If compression is enabled, compress body with zlib and update size fields
3. If key rotation is triggered, embed rotation count in the top byte of the type field
4. Encrypt header with the stream cipher
5. Encrypt body (with rotation split if applicable)

**Inbound:**

1. Accumulate data until 20+ bytes available
2. Decrypt 20-byte header
3. Extract rotation hint from top byte of type field; clear those bits
4. Read body length from decrypted header; accumulate until complete
5. Decrypt body (with rotation split if applicable)
6. If compression is active, decompress body

### Key Rotation

Key rotation provides forward secrecy by periodically re-deriving the cipher key during the session:

```
new_key = MAC(key=current_key, msg=session_key)
```

Rotation is signalled in the **top 8 bits of the packet type field** in the encrypted header. The rotation count (1–63) indicates how many derivation iterations to apply. Most packets carry a rotation count of zero (no rotation).

The sender:
1. Encrypts the header (with rotation count embedded)
2. Encrypts the first 2 bytes of the body
3. Applies key rotation
4. Encrypts the remaining body

The receiver mirrors this sequence exactly to stay in sync.

### Compression

| Name | Description |
|---|---|
| `GZIP` / `ZLIB` | zlib compression with `Z_SYNC_FLUSH` |

Compression is applied **before** encryption on outbound and **after** decryption on inbound. The compressed size replaces both size fields in the 20-byte Hotline header.

---

## Compatibility

If a client does not include `DATA_HOPE_MAC_ALGORITHM` in its login packet (or sends a real login value instead of `0x00`), the server treats it as a standard Hotline login:

- Login and password are decoded from the legacy bitwise-inversion encoding
- No transport encryption is established — all subsequent traffic is plaintext
- The connection proceeds normally

HOPE can be disabled server-side. Clients should be prepared to fall back to the standard login flow.

---

## App ID and App String Conventions

The `app_id` field is a 4-byte value. For macOS applications, this should be the creator code (OSType). For other platforms, use a 4-character abbreviation of the program name.

Known app IDs:

| App ID | Application |
|---|---|
| `HTSt` | HotSocket |
| `HOTS` | HotSocket |
| `HOPE` | HotSocket |
| `HLSR` | hlserver |
| `JNSV` | Janus |

Some clients (gtkhx, hx, GLoarbLine) omit `DATA_HOPE_APP_ID` and `DATA_HOPE_APP_STRING` in their HOPE identification entirely.

The `app_string` field is free-form and should contain at least the application name and version (e.g. `"HotSocket 1.0"`, `"HLServer 0.2.7"`).

HOPE identification is the **only** mechanism in the Hotline protocol for a client to advertise its application name and version. Legacy clients that do not implement HOPE are identified solely by the 16-bit version number in the handshake magic and the Version field (160) in the Login transaction.

---

## Sequence Diagram

```
Client                                          Server
  |                                                |
  |---- Hotline Magic (12 bytes) ----------------->|
  |<--- Server Magic (8 bytes) --------------------|
  |                                                |
  |---- htlcHdrLogin (Identification) ------------>|
  |     login = 0x00                               |
  |     MAC algorithms = [HMAC-SHA1, SHA1, ...]    |
  |     app_id, app_string                         |
  |     ciphers = [RC4, BLOWFISH]  (optional)      |
  |     compression = [GZIP]       (optional)      |
  |                                                |
  |<--- htlsHdrTask (Identification Reply) --------|
  |     session_key (64 bytes)                     |
  |     chosen_algorithm = HMAC-SHA1               |
  |     login_field = HMAC-SHA1 (or empty)         |
  |     server app_id, app_string                  |
  |     cipher = RC4, mode = STREAM  (optional)    |
  |     compression = GZIP           (optional)    |
  |                                                |
  |---- htlcHdrLogin (Authenticated) ------------->|
  |     login = mac(login, session_key)            |
  |     password = mac(password, session_key)      |
  |     nick, icon                                 |
  |     cipher echo (0x0EC1, 0x0EC9)  (optional)   |
  |                                                |
  |  [Server verifies MACs, derives transport keys]|
  |  [encode_key = MAC(password, password_mac)]    |
  |  [decode_key = MAC(password, encode_key)]      |
  |  [Activates cipher + optional compression]     |
  |                                                |
  |<===== ALL FURTHER TRAFFIC ENCRYPTED ==========>|
```
