# Draft: Portable Identity for the Instant Messaging Extension

> **Status:** Draft proposal by Misha Nasledov (hxd-ng). **Not adopted**: nothing here is part of the [Instant Messaging Extension](Capabilities-Messaging.md) unless that document says so. It proposes field IDs `0x061F` and `0x0628`-`0x062B`, capability bit 13 and reason codes 15 and 16, all unassigned at the time of writing. Until it is adopted, they are used only between servers and clients whose operators and users have chosen to try it, and nothing here changes what is asked of a server or client that has not.

An optional addition to the [Instant Messaging Extension](Capabilities-Messaging.md), written in that document's format and linking to its sections. Identity keys are those of [hxd-ng's identity spec](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md): an Ed25519 identity key that a user holds across servers, and device certificates signed by it.

It does two things, for accounts linked to such a key:

- **The account's identity fingerprint travels beside its Login**, on roster entries and discovery results, so a client can keep a row through a rename or a key rotation and recognize the same person on another server.
- **Instant messages between two such friends can be signed, sealed, or both**, so that no server can alter them or write one in the sender's name, or read them.

This version covers friends on one server. Handles from an identity registrar, friends on linked servers, and sealed messages for devices that are offline are left for later and explained under [Possible Follow-ups](#possible-follow-ups).

---

## Portable Identity

### What Does Not Change

- **The bare identity is still the Login.** Roster rows, blocks, queues and authorization stay keyed by Login. The fingerprint travels beside it, never in its place.
- **The [reserved logins](Capabilities-Messaging.md#reserved-logins) and discovery rules apply unchanged**, to the fingerprint as to the Login. A session that proved a key but is logged in as `guest` is still `guest`, and never enters the system. An operator who wants identity users to message gives them accounts of their own.
- **The sender is the session.** The server attributes a message to the Login, and the identity, that the session authenticated as.

### Capability Bit

| Bit | Mask | Name | Meaning |
|---|---|---|---|
| 13 | `0x2000` | `CAPABILITY_IDENTITY` | The client understands the identity field and signed and sealed messages |

A client sets bit 13 when it understands the fields below; it need not have an identity of its own. A server confirms it only when it implements this document, and only alongside `CAPABILITY_MESSAGING` (bit 6).

A client that sets bit 13 on a session that proved, at login, a device key whose certificate allows messages and is at most 512 bytes MUST be able to open sealed messages for that key. Such a session is **sealed-capable**. That is the cost of one bit: a client with such a key that cannot open sealed messages, such as a classic client tunneled under a key, leaves bit 13 off, and so also goes without renames in place and signed messages. Whether a friend's session is sealed-capable needs no bit of its own: [Get User Info (825)](Capabilities-Messaging.md#get-user-info-825) lists a device's certificate only while it has one ([Device Certificates](#device-certificates)).

### The Fingerprint

An account's **fingerprint** is the SHA-256 digest of its 32-byte identity public key ([identity spec section 3.2](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#32-fingerprint)), carried in one new field:

| Field | Notes |
|---|---|
| `DATA_FRIEND_IDENTITY` | The fingerprint of the identity linked to the account, 32 bytes |

- **A server sends it only for an account linked to an identity**, and only where it would show that link to an ordinary user of the server, by its own policy or the account holder's setting. A fingerprint lets anyone recognize the same person on every server where they use the key, so it is the user's to share.
- **Never for an account the server refuses for messaging**: one without `AccessMessaging`, a reserved login, or any other account it treats the same way. Such an account appears nowhere in messaging, and its fingerprint does not either.
- **It is a public-card field**, like `FieldUserName`: sent on entries in every roster state ([Get Roster (800)](Capabilities-Messaging.md#get-roster-800), [Roster Entry (801)](Capabilities-Messaging.md#roster-entry-801)), in [Friend Request (804)](Capabilities-Messaging.md#friend-request-804) and [Presence Changed (809)](Capabilities-Messaging.md#presence-changed-809), and in the results of [Find User (822)](Capabilities-Messaging.md#find-user-822), [User Search (823)](Capabilities-Messaging.md#user-search-823) and [Get User Info (825)](Capabilities-Messaging.md#get-user-info-825), to friends and non-friends alike. It is the hash of a public key and says nothing about whether anyone is signed in.
- It follows the [complete-entry rule](Capabilities-Messaging.md#an-entry-group-is-complete-not-incremental): an entry without it is one whose account shows no identity.
- A server MAY send it to a session that did not negotiate bit 13, which ignores it as an unknown field. Of the rules for receiving it, only the rename rule below depends on the bit.

A client treats the fingerprint as the durable key for a row: two entries with the same fingerprint are the same person, even under different Logins or on different servers, and a client MAY offer to merge local state (aliases, history) across them. The Login remains the address.

### Renames and Rotation

**Rename.** When an account linked to an identity is renamed, a friend's session that negotiated bit 13 receives one Roster Entry (801) for the new Login, carrying `DATA_FRIEND_IDENTITY`, with no `Removed` before it, and MUST rename the row it holds under that fingerprint in place, keeping its alias and local state. Every other session, and every session for an account that shows no identity, receives the removal followed by the fresh entry, as under [Persistence](Capabilities-Messaging.md#persistence). A client that keys rows by Login therefore never sees a Login change without a `Removed`.

**Rotation.** An identity can rotate to a successor key ([identity spec section 8.5](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#85-rotation)), after which the account carries a new fingerprint under the same Login. The server sends a Roster Entry (801) with the new fingerprint to every online friend, and no `Removed`; the account's queue, blocks and receipts follow it unchanged. A client that receives an entry whose Login matches a row it holds and whose fingerprint differs, with no `Removed` for that Login in between, treats it as a rotation: the row keeps its state and takes the new fingerprint. A Login that passes to another account always arrives after a `Removed`, so the two cannot be confused. A client SHOULD tell its user when a friend's fingerprint changes, since on the wire a rotation looks the same as a server substituting a key.

### Lookup by Fingerprint

[Find User (822)](Capabilities-Messaging.md#find-user-822) accepts `DATA_FRIEND_IDENTITY` in place of `DATA_FRIEND_LOGIN`. The reply keeps its shape and always carries `DATA_FRIEND_LOGIN`, which the caller addresses from then on. A request carrying both, or neither, is a failure with no reason code.

- **It never answers `NotMessagingEnabled`.** A fingerprint with no account here, linked to an account the server refuses for messaging, or not shown under the rules above, is `AccountNotFound`, as is a subject that has blocked the caller. A fingerprint is the same on every server, so anyone who knows a person's key could otherwise ask each server which Login it belongs to, and pair a known person with an administrative login.
- The rate limits of Find User apply unchanged, and a lookup by fingerprint counts against them as a lookup by Login does.

[User Search (823)](Capabilities-Messaging.md#user-search-823) does not search fingerprints: nobody types a fragment of a hash.

### Signed and Sealed Messages

A message between two friends whose accounts are linked to identities can be **signed** by the sender's device key, so that no server can alter it or write one in the sender's name, and **sealed**: encrypted to one of the recipient's devices, so that no server can read it. The sealed form is the one the Server Linking Extension's [user keys draft](Server-Link-User-Keys.md#sealed-messages-in-link-private-message-905) uses for private messages across links. This section is what [IM Send (810)](Capabilities-Messaging.md#im-send-810) and [IM Deliver (811)](Capabilities-Messaging.md#im-deliver-811) carry for it, and what the [offline queue](Capabilities-Messaging.md#offline-delivery) does with it; beyond that, the server treats it as opaque.

Each message is sent in one of four forms, the sender's choice:

| Form | IM Send carries | Who can attribute it |
|---|---|---|
| Plain | `DATA_MESSAGE_BODY` | Nobody: the server's word is all there is |
| Signed | `DATA_MESSAGE_BODY` and `DATA_MESSAGE_SIGNED` | Anyone who holds it: a signature is transferable |
| Sealed and signed | `DATA_MESSAGE_SEALED` and `DATA_MESSAGE_SEALED_TO` | The recipient, who can also prove it to others |
| Sealed deniably | `DATA_MESSAGE_SEALED` and `DATA_MESSAGE_SEALED_TO` | The recipient alone: HPKE's auth mode convinces the recipient that the sender's device wrote it, and nobody else, since the recipient's key could have made the same box |

In IM Send and IM Deliver, `DATA_MESSAGE_BODY` is REQUIRED unless the message is sealed.

#### Signed Messages

| Field | Notes |
|---|---|
| `DATA_MESSAGE_SIGNED` | The signed message in detached form, at most 1020 bytes, beside `DATA_MESSAGE_BODY` |

The signed message is a CBOR map in deterministic encoding, signed by the sender device's Ed25519 key under the domain `hl-identity/message/v1`. It covers a random 16-byte message ID, the time it was signed, the recipient's identity fingerprint, the sender's device certificate and the text, and leaves the text out of the detached form, since it travels in `DATA_MESSAGE_BODY`.

- The sender's session MUST have negotiated bit 13 and `CAPABILITY_TEXT_ENCODING`, and proved a device key at login. The signature covers the body as UTF-8, with each CR LF pair and then each lone CR replaced by LF, at most 4096 bytes; a client verifies after doing the same to the body it receives.
- **The message ID is the `DATA_MESSAGE_GUID`.** Deduplication by `(GUID, recipient)` is unchanged.
- A server refuses `DATA_MESSAGE_SIGNED` to a recipient with no linked identity, with `FieldError` text and no reason code.
- **A server MAY check it** before accepting the send: that the signature verifies, that the certificate belongs to the identity linked to the sender's account, that the recipient fingerprint is the one linked to the recipient's, that the ID is the GUID, that its signed time is within a few minutes of the server's clock, and that the signer has not used the ID for a message with other content. A server that checks refuses a send that fails with `BadSignature`. A server that wanted to tamper could strip the signature instead, so the check protects little; what protects users is the client's warning when a signature goes missing ([Security Considerations](#security-considerations)).
- The server stores it with the message and delivers it unchanged beside the body, live and from the offline queue, to every recipient session that negotiated `CAPABILITY_TEXT_ENCODING`. To any other session it sends the body alone, because the text that session receives is no longer the text that was signed.

#### Sealed Messages

| Field | Notes |
|---|---|
| `DATA_MESSAGE_SEALED` | The sealed message, at most 17,152 bytes, in place of `DATA_MESSAGE_BODY` |
| `DATA_MESSAGE_SEALED_TO` | The SHA-256 of the `device_enc` key the message was sealed to, 32 bytes |

The sealed message is either a whole signed message, padded and encrypted with HPKE (RFC 9180) to one recipient device's X25519 key (`device_enc` in its certificate), or the same message without a signature, sealed in HPKE's auth mode under the sender device's own certified `device_enc` key. Its first byte says which.

- A client MUST NOT send a sealed message unless the server confirmed bit 13. A sealed send carries no `DATA_MESSAGE_BODY` and no `DATA_MESSAGE_SIGNED`, since a plaintext body beside it would make sealing it pointless; a send that carries either is a failure with no reason code.
- **A sealed message is for one device.** The server delivers it only to the recipient's sealed-capable sessions that proved the device whose `device_enc` key hashes to `DATA_MESSAGE_SEALED_TO`. The Login's other sessions are not sent it. A sender who wants each of the recipient's live devices to have the message seals one copy per device, each with its own GUID and so its own message ID, and deduplication and receipts work per copy as they do per message.
- **Sealed messages are not queued.** If no such session is live, the send fails with `NoDeviceKeys`, and with `OfflineUndeliverable` when the recipient has no live session at all. The recipient judges a sealed message's age by its own clock and refuses one sealed more than 24 hours before it is opened, as the user keys draft has it, so a queue that holds messages for days would deliver what the recipient then drops. A sending client that is told `NoDeviceKeys` fetches the certificates again; it does not fall back to sending in the clear.
- `MaxMessageBytes` bounds `DATA_MESSAGE_BODY` only. A sealed message is padded so that its size hides its length, and is bounded by its construction instead; a server refuses one over 17,152 bytes with `MessageTooLong`.
- The server MUST NOT alter, re-encode, store or log the contents of `DATA_MESSAGE_SEALED`.

#### Device Certificates

| Field | Notes |
|---|---|
| `DATA_FRIEND_DEVICE_CERT` | A device certificate, at most 512 bytes. **Repeated** |

To seal a message, a sender needs the recipient device's certificate. Get User Info (825) returns, to an accepted friend only, one `DATA_FRIEND_DEVICE_CERT` for each of the subject's devices that has a live sealed-capable session: the certificate the device presented at login ([identity spec section 3.3](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#33-device-certificate)), unexpired, not revoked as far as the server knows, allowing messages, and at most 512 bytes. That a certificate is listed is how a sender knows the device can receive sealed messages now. It is friend-only for the same reason as `DATA_FRIEND_CAPABILITIES`: it changes with the subject's devices and sessions, which a stranger has no business knowing. And, as `DATA_FRIEND_CAPABILITIES` is, the list is withheld from everyone while the subject is `Invisible`: a listed certificate means a live session, so a friend would otherwise learn that an `Invisible` user is online.

**The sending client checks each certificate** against the subject's `DATA_FRIEND_IDENTITY`: that its identity key hashes to the fingerprint, that the identity key's signature on it verifies, that it has not expired, and that it allows messages. It seals only to a certificate that passes, and treats one that fails as absent. A server that handed out bare keys could substitute its own and read everything; a certificate it cannot forge, checked against the fingerprint, is what prevents that.

**The receiving client checks the sender**, in the order the construction gives: the recipient fingerprint, the message's age, that its ID has not been accepted from that sender before, the certificate inside the message against the sender's `DATA_FRIEND_IDENTITY`, and the signature, or for a deniable box that the certificate's `device_enc` key is the one that authenticated it. A message that fails is shown as unverified, never as from that friend.

### Additions to the Tables

Data objects, from `0x061F`, the last ID the extension reserves for a messaging field, and `0x0628`-`0x062F`, past its limits sub-block:

| ID (hex) | Dec | Name | Type | Description |
|---|---|---|---|---|
| `0x061F` | 1567 | `DATA_FRIEND_IDENTITY` | Binary (32) | The fingerprint of the identity linked to an account |
| `0x0628` | 1576 | `DATA_FRIEND_DEVICE_CERT` | Binary (at most 512) | A device certificate, in a Get User Info reply. **Repeated** |
| `0x0629` | 1577 | `DATA_MESSAGE_SIGNED` | Binary (at most 1020) | A signed message in detached form, in 810 and 811 |
| `0x062A` | 1578 | `DATA_MESSAGE_SEALED` | Binary (at most 17,152) | A sealed message, in 810 and 811 |
| `0x062B` | 1579 | `DATA_MESSAGE_SEALED_TO` | Binary (32) | The SHA-256 of the `device_enc` key a sealed message was sealed to, in 810 and 811 |

Reason codes:

| Code | Name | Meaning |
|---|---|---|
| 15 | `NoDeviceKeys` | A sealed message names a device key no live sealed-capable session of the recipient proved, or the recipient has none |
| 16 | `BadSignature` | A signed message does not verify, is not scoped to the recipient, or reuses the message ID of another message by the same signer |

The user keys draft has reasons of the same names in the link's list, as 13 and 14. They are different numbers because 13 and 14 are taken here.

`MessageTooLong` (13) also covers a sealed message over 17,152 bytes, and IM Send's reply may carry `NoDeviceKeys`, `OfflineUndeliverable` or `BadSignature`.

### Security Considerations

- **Accounts without passwords.** An account linked to an identity may log in by proving its key, with no password at all. For such an account the Login being public exposes nothing. The reserved logins rule is unaffected: it is about what those accounts are, not how they authenticate.
- **Fingerprints link people across servers.** Anyone who sees a user's fingerprint can recognize them wherever they use the key, which is why a server sends it only where it would show it to its own users, and never for an account messaging refuses.
- **Sealed messages.** No server can read a sealed message, including the one that carries it. The server still sees who messages whom, when, and roughly how much. A client that seals to a certificate it did not check against the recipient's fingerprint has sealed to whoever supplied the certificate, and the fingerprint is only as good as the server's word: a server that replaces a friend's fingerprint and serves a matching certificate can read what is sealed to it. **On first contact, only comparing fingerprints out of band protects against that.**
- **Signed messages are evidence.** A signature is transferable: whoever holds a signed message, sealed or not, can prove to anyone that the sender's device wrote it. That is what a sender chooses by signing, and why a deniable form exists beside it. Clients SHOULD let the user choose, and SHOULD show which form each message arrived in.
- **Downgrade.** A server can strip a signature or refuse to deliver a sealed message, and a client that silently falls back sends in the clear. This is what protects users, not the server's check. Clients SHOULD remember which friends they have exchanged signed or sealed messages with, and warn before sending to one of them in the clear or when a message from one arrives without its signature.
- **Identifier spoofing.** A server takes `DATA_FRIEND_IDENTITY` from the account it resolved, never from the client, as it does the Login, and a fingerprint in a request is a lookup key, not a claim.

### Client Behaviour

No client change is required: a client that does not set bit 13 sees what it sees today, and is never sent a signed or sealed message it cannot read. A client that supports this document:

1. Treats `DATA_FRIEND_IDENTITY`, where present, as the durable key for a roster row, and applies [Renames and Rotation](#renames-and-rotation).
2. Seals only to device certificates it checked against the recipient's fingerprint, and does not fall back to the clear on `NoDeviceKeys`.
3. Shows a signed or sealed message that fails its checks as unverified, never as from that friend, and follows the advice under Downgrade above.

### Implementation Notes

- hxd-ng implements the identity spec's objects in `crates/hl-identity`, including device certificates and their verification. A server implementing this document need check nothing cryptographic: the signature check is a MAY, and only clients open sealed messages.
- The construction of signed and sealed messages is the one hxd-ng's [end-to-end messages document](https://github.com/mishan/hxd-ng/blob/main/docs/e2e-messages.md) defines for Hotline-ng and server links. One difference: on a link, a detached signed message carries its certificate's SHA-256 in place of the certificate, to fit Link Chat (904), while here it carries the certificate as that document does, so a message read from the offline queue still verifies after its sender has left.

---

## Possible Follow-ups

**Handles.** A registrar's attestation that a key holds a handle, such as `bob@hl.example`, so that people can find each other by something more readable than a fingerprint. Given on the server's word alone it would be a claim nobody can check, so it waits for the same follow-up as in the [user keys draft](Server-Link-User-Keys.md#possible-follow-ups): an attestation carried with it, and a list of registrars the receiving client trusts. Find User (822) and User Search (823) would then accept it as they accept a Login.

**Friends on linked servers.** Ghosts never take part in messaging, so this version covers friends on one server. Carrying signed and sealed messages to a friend on a linked server, as the user keys draft does for classic private messages, needs an exception to the rule that no messaging transaction names a ghost, and a roster across links. That is left deliberately for a later version.

**Sealed messages for offline devices.** Sealing mail for a device that is offline needs a construction that marks a message as mail, so the recipient judges it by when it was queued rather than refusing it as stale, and offline queue rules for a sealed body.
