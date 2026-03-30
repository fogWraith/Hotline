# Hotline Tracker Federation (HTFD)

**Companion specification to [Tracker Protocol v3](tracker-protocol-v3.md)**

---

## Overview

Federation allows trackers to exchange server lists, so a client connected to one tracker can discover servers registered on another. This enables a decentralized network of trackers without requiring servers to register with every tracker individually.

This is an **optional** companion protocol to the v3 tracker protocol. A tracker implementing v3 does not need to support federation.

Federation connections use **TCP on port 5498**, the same port as client listing. The tracker distinguishes federation connections from listing and HTCI connections by the 4-byte magic.

---

## TLV Field Range

HTFD uses TLV field IDs in the range **0x0700–0x07FF**, reserved by the core v3 specification.

---

## Federation Handshake

**Initiating tracker sends** (8 bytes):

| Offset | Size | Field         | Type  | Value      | Description                    |
|--------|------|---------------|-------|------------|--------------------------------|
| 0      | 4    | Magic         | bytes | `"HTFD"`   | `0x48544644` (Hotline Federation) |
| 4      | 2    | Version       | u16   | `0x0001`   | Federation protocol version 1  |
| 6      | 2    | Feature flags | u16   | bitmask    | Federation features supported  |

**Responding tracker sends** (8 bytes): Same format, echoing magic and version with its own feature flags.

---

## Server List Exchange

After the federation handshake, the trackers exchange their server lists:

**Exchange request:**

| Offset | Size | Field        | Type  | Description                                  |
|--------|------|--------------|-------|----------------------------------------------|
| 0      | 2    | Message type | u16   | `0x0100` = full list exchange                |
| 2      | 2    | Field count  | u16   | TLV auth/metadata fields                     |
| 4      | var  | TLV fields   | TLV[] | Tracker identity, auth credentials           |

**Exchange response:** Standard listing response format (same as client listing), containing all non-promoted, non-private servers.

Trackers MAY maintain persistent federation connections and use delta update messages (`DELTA_ADD`, `DELTA_REMOVE`, `DELTA_UPDATE`) for incremental changes:

| Type     | Name             | Payload                                    |
|----------|------------------|--------------------------------------------|
| `0x0010` | `DELTA_ADD`      | Full server record (new server appeared)   |
| `0x0011` | `DELTA_REMOVE`   | Address type + address + port              |
| `0x0012` | `DELTA_UPDATE`   | Full server record (metadata changed)      |

---

## Trust and Identity

Each tracker has a unique identity for federation:

| Field ID | Name              | Type   | Description                              |
|----------|-------------------|--------|------------------------------------------|
| 0x0700   | `TRACKER_ID`      | bytes  | 32-byte unique tracker identifier        |
| 0x0701   | `TRACKER_NAME`    | string | Display name of the tracker              |
| 0x0702   | `TRACKER_DESC`    | string | Tracker description                      |
| 0x0703   | `TRACKER_CONTACT` | string | Operator contact (email/URL)             |
| 0x0704   | `TRACKER_POLICY`  | string | URL to tracker policy/terms              |
| 0x0710   | `FED_SIGNATURE`   | bytes  | Ed25519 signature over the exchange payload |

Trackers SHOULD verify the identity of federation peers. Options:
- **Pre-shared keys:** Both trackers configure each other's `TRACKER_ID` and a shared secret.
- **Public key signatures:** Each tracker has an Ed25519 keypair. The `TRACKER_ID` is the public key. Signed payloads prove origin.

---

## Propagation Rules

| Rule | Description                                                          |
|------|----------------------------------------------------------------------|
| 1    | **Promoted entries are never federated.** `IS_PROMOTED` records are local to the originating tracker. |
| 2    | **Private listings are never federated.** `PRIVATE_LISTING` servers are excluded from exchange. |
| 3    | **Hop limit.** Federated records include a hop counter (max 2). A record received via federation is not re-federated beyond the configured limit to prevent infinite loops. |
| 4    | **Origin tracking.** Each federated record carries the originating `TRACKER_ID`. The receiving tracker stores this and can attribute servers to their source. |
| 5    | **Expiry.** Federated records expire using the same rules as locally registered records. If the upstream tracker stops sending updates, the records are removed. |

---

## Rate Limiting

| Target                   | Recommended Limit       |
|--------------------------|-------------------------|
| Federation connections   | 1 per minute per peer   |
