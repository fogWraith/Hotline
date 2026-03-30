# HOPE ChaCha20-Poly1305 Extension

This document specifies the ChaCha20-Poly1305 AEAD cipher extension for HOPE transport encryption as well as the file transfer encryption. It extends the base [HOPE Secure Login](HOPE-Secure-Login.md) specification.

## Table of Contents

- [Overview](#overview)
- [Cipher Negotiation](#cipher-negotiation)
  - [Cipher Name](#cipher-name)
  - [Cipher Mode](#cipher-mode)
  - [Checksum Field](#checksum-field)
- [Key Derivation](#key-derivation)
  - [HKDF Expansion](#hkdf-expansion)
  - [AEAD Key Derivation](#aead-key-derivation)
- [AEAD Wire Format](#aead-wire-format)
  - [Frame Structure](#frame-structure)
  - [Nonce Construction](#nonce-construction)
  - [Maximum Frame Size](#maximum-frame-size)
- [File Transfer Encryption](#file-transfer-encryption)
  - [File Transfer Key Derivation](#file-transfer-key-derivation)
  - [Per-Transfer Key](#per-transfer-key)
  - [HTXF Protocol Integration](#htxf-protocol-integration)
- [Backward Compatibility](#backward-compatibility)

---

## Overview

ChaCha20-Poly1305 (RFC 8439) is an Authenticated Encryption with Associated Data (AEAD) cipher that provides both confidentiality and integrity in a single primitive. Unlike the existing stream ciphers (RC4, Blowfish/OFB), AEAD mode eliminates the need for a separate checksum mechanism and prevents undetected tampering of ciphertext.

Key differences from stream cipher mode:

| Property | Stream (RC4/Blowfish) | AEAD (ChaCha20-Poly1305) |
|---|---|---|
| Key size | Variable (MAC output) | 256-bit (HKDF-expanded) |
| Authentication | Optional checksum field | Built-in Poly1305 tag |
| Key rotation | Probabilistic per-packet | Counter-based nonces (no rotation needed) |
| Wire format | Byte-stream XOR | Length-prefixed framed |
| Compression | Supported | Not used |

---

## Cipher Negotiation

### Cipher Name

The cipher is identified in `DATA_HOPE_SERVER_CIPHER` (0x0EC1) and `DATA_HOPE_CLIENT_CIPHER` (0x0EC2) using the algorithm list encoding defined by HOPE:

```
<u16:count> <u8:nameLen> <str:name>
```

The canonical name is `CHACHA20-POLY1305`. Implementations should normalize the following variants:

| Variant | Normalized |
|---|---|
| `CHACHA20-POLY1305` | `CHACHA20-POLY1305` |
| `CHACHA20POLY1305` | `CHACHA20-POLY1305` |
| `CHACHA20` | `CHACHA20-POLY1305` |

### Cipher Mode

When ChaCha20-Poly1305 is negotiated, the server sends `DATA_HOPE_SERVER_CIPHER_MODE` (0x0EC3) and `DATA_HOPE_CLIENT_CIPHER_MODE` (0x0EC4) with the value `"AEAD"` (4 bytes, ASCII). This replaces the `"STREAM"` value used by RC4 and Blowfish.

A client that sees `CHACHA20-POLY1305` as the cipher name may also infer AEAD mode from the cipher name alone, but should prefer the explicit mode field when present.

### Checksum Field

When AEAD mode is active, the server sends `DATA_HOPE_SERVER_CHECKSUM` (0x0EC7) and `DATA_HOPE_CLIENT_CHECKSUM` (0x0EC8) with the value `"AEAD"` (4 bytes, ASCII). This indicates that integrity is provided by the AEAD construction itself and no separate checksum is computed.

---

## Key Derivation

### HKDF Expansion

Stream ciphers use MAC-derived keys directly, which are only 16–32 bytes depending on the MAC algorithm (16 for MD5/HMAC-MD5, 20 for SHA1/HMAC-SHA1, 32 for HMAC-SHA256). ChaCha20-Poly1305 requires 256-bit (32-byte) keys. HKDF-SHA256 (RFC 5869) is used to expand shorter MAC-derived keys to the required length:

> **Note:** When `HMAC-SHA256` is the negotiated MAC algorithm, its 32-byte output already matches the required key size. HKDF expansion is still applied for consistency, but no stretching of weaker key material is needed.

```
expanded_key = HKDF-Expand(
    PRK  = HKDF-Extract(salt=session_key, IKM=input_key),
    info = "hope-aead-key",
    L    = 32
)
```

Where:
- `session_key` is the 64-byte server-generated session key from Step 2
- `input_key` is the MAC-derived key (encode_key or decode_key)

### AEAD Key Derivation

The full key derivation for AEAD mode:

```
password_mac  = MAC(key=password_bytes, msg=session_key)
encode_key    = MAC(key=password_bytes, msg=password_mac)
decode_key    = MAC(key=password_bytes, msg=encode_key)

encode_key_256 = HKDF-SHA256(ikm=encode_key, salt=session_key, info="hope-aead-key")
decode_key_256 = HKDF-SHA256(ikm=decode_key, salt=session_key, info="hope-aead-key")
```

Where `MAC` is the negotiated MAC algorithm (must be MD5, SHA1, HMAC-MD5, HMAC-SHA1, or HMAC-SHA256 — INVERSE is not supported with AEAD).

The server uses `encode_key_256` for outbound (server → client) and `decode_key_256` for inbound (client → server). The client uses them in reverse: reads with `encode_key_256`, writes with `decode_key_256`.

---

## AEAD Wire Format

### Frame Structure

In AEAD mode, the XOR stream encryption with key rotation is replaced by length-prefixed AEAD frames:

```
+-------------------+-------------------------------+
| Length (4 bytes)   | Ciphertext + Tag             |
| big-endian uint32  | (variable + 16 bytes)        |
+-------------------+-------------------------------+
```

- **Length**: The size of the ciphertext including the 16-byte Poly1305 authentication tag, encoded as a 4-byte big-endian unsigned integer. The length field itself is not encrypted or authenticated.
- **Ciphertext + Tag**: The output of `ChaCha20-Poly1305.Seal(plaintext)`. The authentication tag is appended by the AEAD construction.

Each Hotline transaction (header + body) is sealed as a single AEAD frame. The plaintext is the complete transaction bytes as they would appear on the wire without encryption.

### Nonce Construction

ChaCha20-Poly1305 uses a 12-byte (96-bit) nonce. The nonce is constructed deterministically from a direction byte and a monotonically increasing counter:

```
Byte:  0        1-3      4-11
      +--------+--------+------------------+
      |  dir   | 0x0000 | counter (BE u64) |
      +--------+--------+------------------+
```

- **dir** (1 byte): Direction indicator
  - `0x00` = server → client
  - `0x01` = client → server
- **padding** (3 bytes): Zero-filled
- **counter** (8 bytes): Big-endian 64-bit unsigned integer, starting at 0 and incrementing by 1 for each frame sent in that direction

The direction byte ensures that the server and client never produce the same nonce even though they share the same key material. Each side maintains separate send and receive counters.

### Maximum Frame Size

Implementations should enforce a maximum ciphertext frame size of 1 MiB (1,048,576 bytes) to prevent memory exhaustion from malformed length prefixes.

---

## File Transfer Encryption

When AEAD mode is active on the control connection, file transfer connections (HTXF) are also encrypted with ChaCha20-Poly1305. Each file transfer uses a unique key derived from the control session's AEAD keys.

### File Transfer Key Derivation

A base key is first derived from both AEAD transport keys:

```
ft_base_key = HKDF-SHA256(
    ikm  = encode_key_256 || decode_key_256,
    salt = session_key,
    info = "hope-file-transfer"
)
```

Where `||` denotes concatenation and the result is 32 bytes (256 bits).

### Per-Transfer Key

Each file transfer is identified by a 4-byte reference number (the HTXF ref). A unique key is derived for each transfer:

```
transfer_key = HKDF-SHA256(
    ikm  = ft_base_key,
    salt = ref_number_bytes,     // 4 bytes, big-endian
    info = "hope-ft-ref"
)
```

The same `transfer_key` is used by both the server and client for the duration of the transfer.

### HTXF Protocol Integration

The HTXF handshake (the `HTXF` magic bytes, reference number, and size fields) is always sent in plaintext on the raw TCP connection. AEAD encryption begins immediately after the handshake completes:

1. Client connects to the file transfer port (control port + 1)
2. Client sends the HTXF handshake in plaintext
3. Server reads and validates the HTXF handshake in plaintext
4. Both sides derive the transfer key from the ref number
5. Both sides initialize ChaCha20-Poly1305 AEAD with the transfer key
6. All subsequent data (FFO headers, fork data, folder actions) is encrypted using the same framed AEAD format as the control connection

File transfer AEAD uses the same frame structure, nonce construction, and maximum frame size as the control connection. The direction byte in the nonce follows the same convention: `0x00` for server → client, `0x01` for client → server.

---

## Backward Compatibility

- Clients that do not support ChaCha20-Poly1305 will not include it in their cipher list. The server will fall back to a mutually supported stream cipher (RC4 or Blowfish).
- Servers that do not support ChaCha20-Poly1305 will ignore it during cipher negotiation and select the next matching cipher.
- The AEAD mode field (`"AEAD"`) is only sent when ChaCha20-Poly1305 is negotiated. Stream cipher connections continue to use `"STREAM"` or omit the mode field entirely.
- File transfer encryption is only active when the control connection uses AEAD mode. Stream cipher connections use plaintext file transfers (or TLS if configured).
- The INVERSE MAC algorithm cannot be used with AEAD mode because HKDF requires cryptographic key material. If only INVERSE is negotiated, transport encryption is disabled regardless of cipher selection.
- `HMAC-SHA256` is the recommended MAC algorithm when using AEAD mode, as its 32-byte output natively matches the ChaCha20-Poly1305 key size.
