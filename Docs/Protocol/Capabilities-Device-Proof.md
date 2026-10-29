# Draft: Device Proof on the Classic Wire

> **Status:** Draft proposal by Misha Nasledov (hxd-ng). **Not adopted**: nothing here is part of the protocol unless another document says so. It uses capability bit 14, transactions 1020 and 1021 and field IDs `0x0420`-`0x0424`, which fogWraith reserved for this draft on 2026-10-29, with transactions 1022-1029 and fields `0x0425`-`0x042F` kept for its growth; they lie outside the [Instant Messaging Extension](Capabilities-Messaging.md)'s ranges, since nothing here depends on messaging. Until it is adopted, they are used only between servers and clients that have chosen to try it.

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This extension lets a classic session on a plain TRTP connection prove that it holds a device key: an Ed25519 key certified by the user's identity key, as [hxd-ng's identity spec](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md) defines them. The client signs a fresh challenge from the server, and from then on the server knows which device the session is.

Two drafts already assume a session can do this. The [messaging identity draft](Capabilities-Messaging-Identity.md)'s first version makes a session sealed-capable only if it "proved, at login, a device key", and the [user keys draft](Server-Link-User-Keys.md) carries a session's fingerprint across links only if it "proved the key at login there, with a proof bound to that server". Today the only ways to do that are an identity login over a WebSocket and a classic session tunneled under one. This is the way for a client that connects over TCP.

## Table of Contents

- [What an Existing Client Sees](#what-an-existing-client-sees)
- [Negotiation](#negotiation)
- [The Objects](#the-objects)
- [Get Device Challenge](#get-device-challenge)
- [Prove Device](#prove-device)
- [When to Prove](#when-to-prove)
- [What It Protects, and What It Does Not](#what-it-protects-and-what-it-does-not)
- [Used By](#used-by)
- [Additions to the Tables](#additions-to-the-tables)
- [Server Behavior](#server-behavior)
- [Client Behavior](#client-behavior)
- [Implementation Notes](#implementation-notes)

---

## What an Existing Client Sees

Nothing. A client that does not set the bit is never sent these transactions or fields, and a server that does not confirm it is never sent them. A session that proves a device is otherwise the session it was: same account, same access, same user list entry. What a proof unlocks is up to the documents that use it ([Used By](#used-by)).

## Negotiation

- A new capability bit, `CAPABILITY_DEVICE_PROOF`, in `DATA_CAPABILITIES`. A client advertises it at login; a server confirms it only when it implements this document, has a server key (identity spec section 3.7), and is configured with at least one host to answer as on the classic wire.
- It does not depend on any other bit. In particular it does not need `CAPABILITY_MESSAGING` (bit 6) or `CAPABILITY_IDENTITY` (bit 13).
- A client MUST NOT send either transaction unless the server confirmed the bit.

## The Objects

The two objects are those of the challenge binding in hxd-ng's [authentication spec, section 6.2](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-auth.md#62-challenge-binding), deterministic CBOR signed with Ed25519, under signature domains of their own:

- a **TRTP server challenge**, domain `hl-identity/server-challenge-trtp/v1`, with the keys of that section's server challenge: `v = 1`, the 32-byte `challenge`, `server_key`, `host`, `expires`, and optionally `next`, the hash of the server key that will follow it;
- a **device proof**, domain `hl-identity/device-proof/v1`, with the keys of its login proof: `v = 1`, `challenge`, `server_key`, `host`, `device` (the device key) and `time` (the signer's clock), and on a TLS 1.3 connection `exporter` ([What It Protects](#what-it-protects-and-what-it-does-not)).

The domains differ from the WebSocket login's so that neither object can be spent on the other wire. Keys alone could not do it: a checker that ignores keys it does not know would accept a device proof as a login proof if the domains were the same.

**The host** is the address the client dialed, in section 6.2's form of a host (lowercase, A-labels, bracketed IPv6), with the port always included, since clients differ in whether they show a conventional one. It is ASCII, so it is the same under Text Encoding, and at most 260 bytes: 253 for a name, a kept trailing dot and `:65535` come to 260. A TRTP server challenge is about 222 bytes beside its host and a device proof about 264 with `exporter`, so each fits in 576.

## Get Device Challenge

Request/reply.

**Request fields:** `DATA_PROOF_HOST`, the host the client dialed (REQUIRED).

**Reply fields:** `DATA_SERVER_CHALLENGE`, a TRTP server challenge signed for that host.

- The server answers only for a host it was configured to answer as on the classic wire, and only for one whose port is that of the listener the request arrived on, never whatever a client sends; otherwise, or with no server key, it replies with an error. A plain listener and a TLS listener are thus never answered for each other's host.
- A session holds one challenge at a time. A new request replaces it, and it expires at its `expires`, RECOMMENDED 60 seconds after it is issued as on the WebSocket port, and is spent on use.
- **The client checks it** as section 6.2 has a client check a server challenge, under this domain: the signature under the server key it expects for that host, the host it dialed, and that it has not expired, within the client's clock-skew tolerance. It signs nothing for a challenge that fails.
- **The server key it expects** is the server's identity key (identity spec section 3.7), never a link server key. A client that has none for the host pins the one it is shown on first contact, as hxd-ng's [server key handover spec, section 5.1](https://github.com/mishan/hxd-ng/blob/main/docs/server-key-handover.md#51-pinning) has it. When a later challenge names another key, the client follows the handover chain from the server's discovery document where one answers (authentication spec section 5.1), and otherwise treats it as that spec's changed key, warning the user rather than signing.

## Prove Device

Request/reply.

**Request fields:** `DATA_DEVICE_PROOF` (REQUIRED), a device proof for the session's challenge; `DATA_DEVICE_CERT` (REQUIRED), the device's certificate (identity spec section 3.3); `DATA_USER_CARD`, the identity's card (section 3.4), REQUIRED unless the server holds a card for that identity from an earlier login (section 5.5).

**Reply fields:** none on success.

The server checks, failing on the first error, and spends the challenge whatever follows:

1. the proof decodes, and its signature verifies with its `device` key;
2. its `challenge` is the session's and has not expired, its `server_key` is the server's, and its `host` is the challenge's;
3. its `time` is within the server's clock-skew tolerance;
4. its `exporter` is present if and only if the server's side of the connection is TLS 1.3, and is then the server's own exporter value for the connection;
5. the certificate is for `device`, is signed by its identity key, is within its validity, allows login, is at most 512 bytes, and is not revoked as far as the server knows;
6. the identity, with the card carried or held, is admitted by the server's policy for an identity login (identity spec [section 5.2](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#52-verification)) with an outcome other than refusal or `guest`; a proof never changes the session's access, so probation's mask does not apply to it;
7. the identity is the one the account is linked to, and the account accepts identity login; or the request links it (below).

**Errors** use the ordinary error reply, a non-zero error code with a message for the user in `DATA_ERROR` (100), as other extensions outside messaging do. The message is the same for every failure except step 3, which says the device's clock is off, so a user can fix that one, and so that the answer says nothing of why else a proof failed: a revoked device, an admission refusal and a wrong identity read alike. Clients show the message and MUST NOT parse it.

- **Proving can link.** A server MAY let a Prove Device link an account that is linked to no identity, when it carries the identity's card and the certificate has the identity spec's `manage` capability, under the rules the identity spec gives for linking an existing classic account ([section 8.2](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#82-linking-an-existing-classic-account)): the account allows it, the session logged in with the account's password, and the identity is linked to no other account on the server. A server MUST link only on a session encrypted by TLS 1.3, with the exporter checked, or by HOPE with a cipher, since on any other someone who rode the session or sniffed its password could link the account to an identity of their own. Otherwise the request fails as any other, and nothing is linked.
- **One device, once.** On success the session is device-proved, with that certificate, for its life. Prove Device on a session already device-proved, by this or any other means, such as a tunnel, replies with an error.
- **An account with no password** is not linked by a proof (identity spec section 8.2), and once linked is refused the classic Login (107) that comes before, so it proves its device over a WebSocket instead.

## When to Prove

A client SHOULD prove after the login reply and before it joins the user list: before Agreed (121), or, for a client that finishes its login with Set Client User Info (304), before that. A server may announce a session whose Login carries a name at once, so a client that means to prove before it joins sends no User Name (102) or User Icon ID (104) in its Login, and gives them in Agreed (121). A server that confirms the bit MUST accept both transactions between its login reply and the session's Agreed or 304. The session is then device-proved from its first announcement, which is when the user keys draft carries a fingerprint and when other documents first describe the session to anyone.

A proof made after the session joined is valid on this server, but a server MUST NOT add a fingerprint to a session it has already exported over a link, since the user keys draft has a session's fingerprint stay what it was for the session's life.

## What It Protects, and What It Does Not

- **Bound to the server.** The challenge and the proof both name the server key and the host, so a proof made for one server cannot be spent at another, as section 6.2 has it.
- **Bound to the connection on TLS 1.3.** The proof then carries `exporter`, the TLS 1.3 exporter value (RFC 8446 section 7.5) under the label `EXPERIMENTAL-hl-identity-device-proof`, as the Server Linking Extension's key proof uses an `EXPERIMENTAL-` label, with an empty context, 32 bytes, so a proof cannot be carried onto another connection. The early exporter MUST NOT be used, and neither transaction is sent as 0-RTT early data. TLS 1.2 has an exporter (RFC 5705), but it is bound to the connection only with extended master secret (RFC 7627), so a proof on TLS 1.2 carries none and is bound as on a plain connection. A server behind a proxy that terminates TLS has no exporter, and treats its sessions as plain.
- **Not bound on a plain connection.** There, someone in the middle could ride a proved session. They still sign nothing: what a proved device makes, a signed line, a sealed message, a friendship act, is made with a key that never leaves the device, so riding the session gains what its password already gives, linking aside, which the rule above keeps off such a session. A server MAY refuse Prove Device on a session that is not encrypted, by TLS of either version or by HOPE with a cipher; HOPE's password MAC alone does not count.
- **Revocation is checked at the proof,** and a document that unlocks something for a proved session says whether it checks again at each use.

## Used By

Each of these MAY treat a session proved this way as one that proved its key at login, with one change of wording each:

- **[Messaging identity draft](Capabilities-Messaging-Identity.md):** in its first version, both places that require a device proved at login, the sealed-capable rule ("proved, at login, a device key") and the signed-message rule ("proved a device key at login"), gain "or by [Prove Device](Capabilities-Device-Proof.md#prove-device)". Its second version already counts a session proved this way. The certificate must also allow messages, as that draft requires, and a session receives sealed messages only with that draft's bit 13 as well; this extension alone says nothing of what a client can open.
- **[User keys draft](Server-Link-User-Keys.md#the-fingerprint):** "a session that proved the key at login there, with a proof bound to that server" gains "or by Prove Device before it joined the user list", since a device proof names the host and the server key.

## Additions to the Tables

Capability bit:

| Bit | Mask | Name | Description |
|---|---|---|---|
| 14 | `0x4000` | `CAPABILITY_DEVICE_PROOF` | The session can prove a device key with Get Device Challenge and Prove Device |

Transactions:

| ID | Name | Type | Description |
|---|---|---|---|
| 1020 (`0x03FC`) | Get Device Challenge | Request/reply | Get a TRTP server challenge for the host the client dialed |
| 1021 (`0x03FD`) | Prove Device | Request/reply | Prove a device key, and optionally link an identity |

Fields:

| ID | Name | Type | Description |
|---|---|---|---|
| `0x0420` | `DATA_PROOF_HOST` | String (at most 260) | The host the client dialed |
| `0x0421` | `DATA_SERVER_CHALLENGE` | Binary (at most 576) | A TRTP server challenge |
| `0x0422` | `DATA_DEVICE_PROOF` | Binary (at most 576) | A device proof |
| `0x0423` | `DATA_DEVICE_CERT` | Binary (at most 512) | A device certificate (identity spec section 3.3) |
| `0x0424` | `DATA_USER_CARD` | Binary (at most 16384) | A user card (identity spec section 3.4), only to link |

`DATA_DEVICE_CERT` is the messaging identity draft's `DATA_FRIEND_DEVICE_CERT` in shape, with an ID of its own because this draft does not depend on messaging.

## Server Behavior

- Rate-limit both transactions per account and per address, so that reconnecting earns nothing. Each Get Device Challenge costs a signature by the server key, and each Prove Device the checks of the proof, the certificate and the card, which can reach a registrar; a session's one challenge bounds only what challenges hold. A server MAY refuse further Prove Device on a session after a failure.
- Configure the classic host names apart from any WebSocket listener's, since the two are reached at different addresses.
- Keep what a session proved for the session's life, and drop it with the session.

## Client Behavior

- Prove only on a server that confirmed the bit, with the address actually dialed, and only after checking the challenge.
- Prove before joining ([When to Prove](#when-to-prove)).
- Sign only by the client's own clock, and show the server's message, which says when the clock is the cause.
- Treat a failure as final for the session; try again on the next connection, not in a loop.

## Implementation Notes

hxd-ng implements the WebSocket login's objects in its `hl-identity` crate, will add these beside them, and will be the reference implementation of this draft. Its docs: [identity spec](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md), [authentication spec](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-auth.md).
