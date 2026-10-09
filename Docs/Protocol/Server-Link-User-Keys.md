# Draft: User Keys for the Server Linking Extension

> **Status:** Draft proposal by Misha Nasledov (hxd-ng). **Not adopted**: nothing here is part of the [Server Linking Extension](Capabilities-Server-Link.md) unless that document says so. It proposes field IDs `0x0642`-`0x0644` and `0x0646`, transaction 915, link feature bit 4 and reason code 13, all unassigned at the time of writing (the fields and the transaction from the extension's reserved ranges).

An optional addition to the [Server Linking Extension](Capabilities-Server-Link.md), written in that document's format and linking to its sections. User keys are those of [hxd-ng's identity spec](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md): an Ed25519 identity key that a user holds across servers, and device certificates signed by it.

It does two things, and keeps to the points listed for user keys under the extension's [Future Work](Capabilities-Server-Link.md#future-work):

- **A user's identity fingerprint travels in their user group**, stated by their home server, so servers and clients across the network can tell that sessions belong to the same key.
- **Private messages between two sessions that both hold keys can be end-to-end encrypted across links**, with the recipient session's device certificate fetched over the link itself.

Broadcasting key bans across the network, and handles from an identity registrar, are left for later and explained under [Possible Follow-ups](#possible-follow-ups).

---

## User Keys

### The Fingerprint

A user's **fingerprint** is the SHA-256 digest of their 32-byte identity public key ([identity spec section 3.2](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#32-fingerprint)). Shown to people as lowercase Crockford base32 without padding, it may be shortened to its first 8 characters in a user interface.

The [user group](Capabilities-Server-Link.md#the-user-group) gains one optional field:

| Field | Required | Notes |
|---|---|---|
| `DATA_LINK_USER_KEY` | No | The user's identity fingerprint, 32 bytes |

- **Only the home server adds it, and only for a session that proved the key at login there**: an identity login ([identity spec section 5](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#5-authentication)), or a classic session tunneled under one (section 8.3). A session that logged in to an account linked to a key, with only its password (section 8.6), has not proved the key and carries none.
- **Only where an ordinary local client would see it.** A home server exports the fingerprint only if it would show it to an ordinary, unprivileged local client, by its own policy or the user's setting, under the same rule as [Link User Info](Capabilities-Server-Link.md#link-user-info-906). A fingerprint lets anyone in the network recognize the same person on every server, so it is the user's to share.
- The field carries the key the session proved at login, and it does not change for the life of the session. A user who rotates to a new key logs in with it, and that new session carries the new fingerprint.
- **It is the home server's statement**, and receivers take it as such, which is as far as the [trust model](Capabilities-Server-Link.md#trust-model) goes. No receiver can check that the session proved the key.
- Every other server passes it on unchanged under [Relaying Fields](Capabilities-Server-Link.md#relaying-fields), including servers that implement nothing in this document, provided they implement the revision of the extension that added Relaying Fields. A relay from before that revision strips the field, and the users behind it appear without a fingerprint.

It costs a user group 36 bytes of the 1024 its fields outside the baseline may use.

### Presenting Keys

A receiving server MAY show a ghost's fingerprint where it shows other details about a user: in the [user info](Capabilities-Server-Link.md#user-info) text, after the line naming the home server, and to clients that understand identity. It MUST NOT change a ghost's display name because of it, under [Display Name](Capabilities-Server-Link.md#display-name).

### Key Bans

User keys are identity, not moderation: a key is free to make, so refusing one does little against someone who makes another. A [network ban](Capabilities-Server-Link.md#link-ban-908) of a key holder is recorded by their home server against whatever its own bans use, which may be the key. Any other server may refuse a key it has seen at its own door, as its own ban; hiding ghosts by fingerprint at one server is a server-scoped ban, which the extension already lists as Future Work. This document defines no transaction that tells other servers about a key ban.

### Encrypted Private Messages

When the sending session and the receiving session both proved keys and both carry fingerprints, and their clients and home servers support it, a private message between them can be encrypted end to end. Every server on the path, the two home servers included, carries it without being able to read it. Two things cross the link: the recipient session's **device certificate**, fetched before sending, and the **sealed message**, which carries the sender's certificate inside it.

A private message, like [Link Private Message (905)](Capabilities-Server-Link.md#link-private-message-905) itself, goes to one session: one ghost, on one device. Everything below is therefore per session. A person with several devices online appears as several ghosts, and a message to one of them is readable on that device only. **The device behind a keyed session can change** (a session resumed from another device, for example), and a ghost's ID can pass to another session once the extension's quarantine ends, or, after its home server restarts, stand for another user with the same name and icon on another device, since a receiver may re-use a ghost for them ([Interruption and Resynchronisation](Capabilities-Server-Link.md#interruption-and-resynchronisation)). So a sealed message names the device key it was sealed to, and the home server refuses one that no longer matches (below).

How a client asks its own server for a ghost's certificate, and how it sends and shows a sealed message, is the client protocol's business. hxd-ng's Hotline-ng wire will carry it first. On the classic wire it belongs with the identity amendments proposed for the [Messaging extension](Capabilities-Messaging.md), whose rule that no messaging transaction names a ghost would need a matching exception.

#### Link Device Keys (915)

Request/reply, [relayed](Capabilities-Server-Link.md#routing) hop by hop to the target's home server like [Link User Info (906)](Capabilities-Server-Link.md#link-user-info-906), and only over links that negotiated `LINK_FEATURE_USER_KEYS`. Fetching the certificate over the link is what keeps it private: the requester's server never learns the home server's address, and nothing outside the network learns that the servers are linked.

**Request fields:** `DATA_LINK_USER_ID` (REQUIRED, the requesting user, as the sender identifies them), `DATA_LINK_TARGET_ID` (REQUIRED, the target, as the receiving server identifies them).

**Reply fields:** `DATA_LINK_DEVICE_CERT`, the certificate of the device the target session currently proves; or a failure with `DATA_LINK_REASON`: `RefusedFields`, `FeatureNotNegotiated`, `UnknownUser`, `Excluded`, `RefusesMessages`, `NoDeviceKeys`, `RateLimited` or `Unreachable`.

A server receiving a Link Device Keys request handles it in the order of [Link Private Message](Capabilities-Server-Link.md#link-private-message-905):

1. it applies [Relaying Fields](Capabilities-Server-Link.md#relaying-fields), answering `RefusedFields` for a request it must drop;
2. it answers `FeatureNotNegotiated` if the request arrived over a link that did not negotiate `LINK_FEATURE_USER_KEYS`;
3. it checks that the requester is a current ghost from that link and the target a user exported over it, and answers `UnknownUser` otherwise;
4. if the target is a ghost here, it forwards the request over the link the ghost came from, with both IDs translated, or answers `FeatureNotNegotiated` if that link did not negotiate the feature, and answers with the reply it receives;
5. if the target is local, it answers `Excluded` if either party is [excluded](Capabilities-Server-Link.md#exclusion) at the other's home server as far as it can tell, `RefusesMessages` if the target refuses private messages, and `NoDeviceKeys` unless the target's user group carries a fingerprint (a certificate reveals it, so it is never sent for a user whose fingerprint is not exported) and the target session currently proves a device whose certificate is unexpired, not revoked as of this request, at most 512 bytes, and allows messages (identity spec section 3.3, the `caps` bit for messages), and the session's client can receive sealed messages (a classic client tunneled under a key cannot, for example). Otherwise it answers with that certificate.

Link Device Keys is rate-limited per ghost and per link, like [Link Private Message](Capabilities-Server-Link.md#limits). The originating server refuses it locally, without sending anything, when the requester is not exported over that link, as it does for 905. Link Device Keys does not need `LINK_FEATURE_PRIVATE_MESSAGES` on its path, so it can succeed where the 905 that follows fails with `FeatureNotNegotiated`; a client then reports that private messages cannot reach that user.

**The client checks the certificate** against the fingerprint in the target's user group: that the certificate's `identity` key hashes to the fingerprint, that the identity key's signature on the certificate verifies, that it has not expired, and that it allows messages. It encrypts only to a certificate that passes.

A certificate is signed and nobody on the path can trim it, so everything in it crosses the network: its expiry, and the device's `name` label if it has one. **A certificate that allows messages SHOULD leave `name` out**, and a client MUST NOT use one larger than 512 bytes for sealed messages, which a certificate without a label never approaches.

#### Sealed Messages in Link Private Message (905)

A sealed message is sent as an ordinary [Link Private Message (905)](Capabilities-Server-Link.md#link-private-message-905), with one more field:

| Field | Notes |
|---|---|
| `DATA_LINK_SEALED` | The message, its quote, its options and the sender's proof, encrypted to the recipient session's device |
| `DATA_LINK_SEALED_TO` | The SHA-256 of the recipient device's `device_enc` key the message was sealed to, 32 bytes |

The construction is defined by hxd-ng's [end-to-end messages document](https://github.com/mishan/hxd-ng/blob/main/docs/e2e-messages.md): HPKE (RFC 9180) to the recipient device's X25519 key, with the plaintext signed by the sender device's Ed25519 key. This document fixes what the construction must provide, and its size:

- **It is encrypted to the recipient's `device_enc` key**, from the certificate Link Device Keys returned for the target session.
- **It proves the sender.** Inside the encryption, it carries the sender session's device certificate, and a signature by that device's key over a random 16-byte message ID, the time it was sealed, the recipient's identity fingerprint, the message, the quote and the options. The fingerprint, not the device key, because a certificate does not prove its owner holds its encryption key: someone could issue a certificate under their own identity that reuses another person's key, and the fingerprint keeps a message sealed to it from being passed to that key's real owner as one meant for them. The recipient's client checks the sender's certificate against the fingerprint of the sending ghost, as its own server presents it, exactly as above, and verifies the signature. A message that fails is shown as unverified, never as from that user. Without this, anyone holding the recipient's certificate, which any server on the path can fetch, could seal a message in another user's name.
- **It resists replay, by the recipient's clock.** The recipient's client refuses a message sealed more than 24 hours before its own clock's now, or more than ten minutes after, and drops a message ID from that sender it has already accepted, keeping accepted IDs, across restarts, for as long as a message could still be fresh. No server's clock enters it, the recipient's home server's included, since that server is one of the parties a sealed message keeps out. The 24 hours is how long a message can be held for a detached session; the home server still passes the time it received the message, which the client shows and checks the sender's certificate against.
- **It hides length.** The plaintext is padded to one of a few fixed sizes, since every server on the path sees the size of the sealed field.

`DATA_LINK_SEALED` is at most 17,152 bytes: a message and a quote of up to 8192 bytes each, the sender's certificate of up to 512 bytes, and 256 bytes for the key exchange, the signature, the message ID, the time, the recipient's fingerprint, the encoding, the authentication tag and padding. With its field header that is 17,156 bytes, and `DATA_LINK_SEALED_TO` 36 more, leaving 216 of the 17,408 that [Relaying Fields](Capabilities-Server-Link.md#relaying-fields) allows 905 for any other extension.

- **`FieldData` carries a placeholder**, because 905 requires it, and so that a recipient whose session cannot open the message still has something to read. The sending server sets it to the fixed text `[Encrypted message]`, and it MUST NOT carry any of the message. `FieldQuotingMsg` MUST be absent: the quote is inside the sealed message. A receiving client ignores `FieldOptions` on a sealed message and uses the options inside it, which nobody on the path can change.
- Relaying servers pass the field on unchanged, under Relaying Fields, whether or not they implement this document.
- **The home server delivers** the sealed message to the target session through its own client protocol, with the time it received it. A sender's client encrypts only after Link Device Keys succeeded for that same target. If the network's paths change in between and a relay from before Relaying Fields comes onto the path, the sealed field is stripped and the recipient sees only the placeholder; a sender cannot know that its message arrived sealed.
- **The home server checks `DATA_LINK_SEALED_TO`** against the `device_enc` key of the target session's current certificate, and answers `NoDeviceKeys` if they differ, if the field is missing beside a sealed field, or if the target can no longer receive sealed messages. A sending client that is told so fetches the certificate again; it does not fall back to sending in the clear.
- The ordinary checks of 905 apply unchanged: the sender must be exported over the link, the target must be a user exported over it, exclusions and refusals of private messages apply, and rate limits count a sealed message like any other.

### Link Feature

| Bit | Mask | Name | Meaning |
|---|---|---|---|
| 4 | `0x0010` | `LINK_FEATURE_USER_KEYS` | Link Device Keys (915) crosses the link |

The fingerprint and the sealed field need no feature: they are fields of existing transactions, and [Relaying Fields](Capabilities-Server-Link.md#relaying-fields) carries them across servers that implement it. Only the new transaction needs every link on its path to support it.

### Additions to the Tables

Data objects:

| ID (hex) | Dec | Name | Type | Description |
|---|---|---|---|---|
| `0x0642` | 1602 | `DATA_LINK_USER_KEY` | Binary (32) | A user's identity fingerprint, in their user group |
| `0x0643` | 1603 | `DATA_LINK_DEVICE_CERT` | Binary (at most 512) | A device certificate (identity spec section 3.3), in a Link Device Keys reply |
| `0x0644` | 1604 | `DATA_LINK_SEALED` | Binary (at most 17,152) | A sealed message, in 905 |
| `0x0646` | 1606 | `DATA_LINK_SEALED_TO` | Binary (32) | The SHA-256 of the `device_enc` key a sealed message was sealed to, in 905 |

Transactions:

| ID | Hex | Name | Pattern | Description |
|---|---|---|---|---|
| 915 | `0x0393` | Link Device Keys | Request/reply | Fetch the device certificate of one session from its home server |

Reason codes:

| Code | Name | Used in | Meaning |
|---|---|---|---|
| 13 | `NoDeviceKeys` | 905 and 915 replies | The target session has no device certificate that allows messages, its client cannot receive sealed messages, or (905) the message was sealed to another device key. (The Messaging amendment's reason of the same name is in a different list.) |

**Relaying Fields for 915.** Link Device Keys falls under [Relaying Fields](Capabilities-Server-Link.md#relaying-fields) in both directions, with these baselines; any other field counts toward the 1024-byte bound:

| Transaction | Baseline fields |
|---|---|
| Link Device Keys (915) request | `DATA_LINK_USER_ID`, `DATA_LINK_TARGET_ID` |
| Link Device Keys (915) reply | `DATA_LINK_REASON`, `FieldError`, `DATA_LINK_DEVICE_CERT` |

This needs a small change in the extension, which today lists by number the transactions Relaying Fields applies to and fixes baselines only for those: *Relaying Fields also applies to a relayed transaction a later document defines, with the baseline that document gives it.* Its reason table would also list `RefusesMessages` for 915 replies as well as 905.

### Security Considerations

- **What end to end means here.** No server can read a sealed message, its home servers included, and no server can seal one in another user's name, unless it also replaces that user's fingerprint. What stops a server from substituting its own device certificate, the recipient's or the sender's, is the fingerprint, and the fingerprint is only as good as the home server's word and the honesty of every relay in between, since user groups are not signed. A server that actively replaces a fingerprint and serves a matching certificate can read what it is sent, or seal messages in that user's name, and so can any server on the path that does so. **On first contact, only comparing fingerprints out of band protects against that.**
- **Downgrade.** Any server on the path can make encryption fail, by stripping a fingerprint or refusing Link Device Keys, and a client that silently falls back sends in the clear. Clients SHOULD remember contacts by fingerprint, together with the name and home server they were seen under; warn when a remembered contact appears without a fingerprint or with a different one; warn before sending unencrypted to someone they have exchanged sealed messages with; show for each message whether it was sealed; and seal a reply only to a certificate matching the fingerprint the message it answers was verified under, since a ghost's ID can pass to someone else.
- **Passive reading is what this stops**, and that covers most of the risk on a network: operators who log, a relay that is breached, chat logs copied off a disk. None of them can read a sealed message.
- **Metadata still crosses.** Every server on the path sees who sent a message to whom, and when. Padding hides the exact length.
- **Revocation.** A client cannot tell whether a device has been revoked. It relies on the home server, which answers Link Device Keys only with a certificate not revoked as of the request. A sender's certificate inside a sealed message is checked by nobody but the recipient. The recipient requires the sender's certificate to be unexpired by its own clock as well as when the message was sealed, so an expired key cannot seal a message dated before its expiry. But that does not limit what a stolen device key can do before it expires: whoever holds one can seal new messages in that user's name, under their real fingerprint, until then. A recipient may also ask its own server for the sender's current certificate and compare keys, but that helps only while the sender is online, and it is not a revocation check: the answer comes from whichever home server the sender logged in to, and a thief logs in where the revocation has not been seen. Short certificate lifetimes are the defense, as they are everywhere else in the identity spec.
- **Certificates cross the network** in full, as above, but only the certificate of a device already online as a ghost, and only to a requester who could message that session.
- **No addresses cross.** Certificates are fetched over the link, never from the home server directly, so the network's membership stays private, as the extension's [Security Considerations](Capabilities-Server-Link.md#security-considerations) require.
- **Fingerprints link people across servers.** Anyone who can see a user's fingerprint can recognize them on every server in the network, which is why exporting it follows what the home server would show its own users.

### Client Behaviour

No client change is required: a client that knows nothing of user keys sees ghosts as before, and a sealed message as `[Encrypted message]`. A client that supports user keys MAY show fingerprints, and follows the advice under Downgrade above.

### Implementation Notes

- hxd-ng implements the identity spec's objects in `crates/hl-identity`, including device certificates and their verification. A server implementing this document checks nothing cryptographic: only the clients do.
- Device certificates are at most 4 KiB by the identity spec, but a typical one is under 300 bytes.

---

## Possible Follow-ups

**Key ban notices.** A notification relayed across the network saying that a home server banned a key, so that other servers' operators could choose to refuse it too. It needs a way to lift a ban as well as make one, a bound on how many a server must keep, and an answer to whether one server's ban should reach servers that never linked with it. A key is also cheap to replace, so a notice does little until registrars make keys costly to obtain.

**Handles.** A registrar's attestation that a key holds a handle, such as `bob@hl.example`, carried in the user group so that people can be told apart by something more readable than a fingerprint. Attestations are signed by the registrar, so unlike the fingerprint, a receiver could check one. It needs the receiving server to know which registrars it trusts, and it is larger than a user group's budget comfortably allows today.

**Messages to a person, not a session.** Sealing one message for all of a person's devices, and holding it for devices that are offline, needs the person's device list and an offline queue across links, neither of which the extension has.
