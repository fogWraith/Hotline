# Draft: Portable Identity for the Instant Messaging Extension

> **Status:** Draft proposal by Misha Nasledov (hxd-ng), version 2. **Not adopted**: nothing here is part of the [Instant Messaging Extension](Capabilities-Messaging.md) unless that document says so. Version 1 proposed field IDs `0x061F` and `0x0628`-`0x062B`, capability bit 13 and reason codes 15 and 16, which the extension reserved for this draft on 2026-10-25. Version 2 adds the [portable friendships](#portable-friendships) below, and with them a capability bit, fields, transactions and reason codes, all shown here as placeholders for the extension to assign ([Additions to the Tables](#additions-to-the-tables)). It is proposed together with the [Device Proof extension](Capabilities-Device-Proof.md), which it cites. Until it is adopted, all of it is used only between servers and clients whose operators and users have chosen to try it, and nothing here changes what is asked of a server or client that has not.

An optional addition to the [Instant Messaging Extension](Capabilities-Messaging.md), written in that document's format and linking to its sections. Identity keys are those of [hxd-ng's identity spec](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md): an Ed25519 identity key that a user holds across servers, and device certificates signed by it.

It does four things, for accounts linked to such a key:

- **The account's identity fingerprint travels beside its Login**, on roster entries and discovery results, so a client can keep a row through a rename or a key rotation and recognize the same person on another server.
- **Instant messages between two such friends can be signed, sealed, or both**, so that no server can alter them or write one in the sender's name, or read them.
- **A friendship can belong to the two people rather than to one server.** Accepting a friend request makes a friendship signed by both people's devices, which each client keeps and can bring to another server, where it becomes an accepted roster entry with no new request once the friend has brought it there too. Ending it is a signed withdrawal that reaches every server the friendship was brought to.
- **A device proved by any means counts.** A session that proved a device by an identity login, by a tunnel under one, or with the [Device Proof extension](Capabilities-Device-Proof.md) is proved for this draft, so a classic client on plain TRTP can be sealed-capable and make friendships.

Handles from an identity registrar, presence and messaging with friends on linked servers, and sealed messages for devices that are offline are left for later and explained under [Possible Follow-ups](#possible-follow-ups).

---

## What an Existing Client Sees

Everything here is opt-in. A client that sets neither bit 13 nor `CAPABILITY_FRIENDSHIPS`, sends none of the fields below and sends none of the transactions below is answered exactly as the extension answers it, and is sent nothing but what it ignores as an unknown field under the extension's rules, such as `DATA_FRIEND_IDENTITY`; none of version 2's fields reaches it ([Capability Bits](#capability-bits)). A client that sets bit 13 and not `CAPABILITY_FRIENDSHIPS` sees version 1.

## Changes from Version 1

- **Portable friendships** ([below](#portable-friendships)), behind a capability bit of their own, `CAPABILITY_FRIENDSHIPS`, since bit 13 means version 1, whose clients send no offers and would show an upgrade offer as a request from someone already a friend.
- **"Proved at login" becomes "proved"** ([Proving a Device](#proving-a-device)). Version 1 has a sealed-capable session prove its device at login, and defines no way to do that over plain TRTP. A session proved by any means the identity spec or the [Device Proof extension](Capabilities-Device-Proof.md) gives counts, from the moment it is proved.
- **`BadSignature` widened** to friendship objects.
- **A server checks what it is given.** Version 1's server "need check nothing cryptographic". A server that implements portable friendships checks every friendship object it is given.
- **`DATA_FRIEND_IDENTITY` to a hidden link's friend.** Version 1 sends the fingerprint only where the server would show the link to an ordinary user. A user who presents a friendship consents to be found by that one friend ([Hidden links](#hidden-links)).
- **An offer hands a stranger a device certificate**, which version 1 gives friends alone ([Making One](#making-one)).
- **A server MAY key its records by fingerprint.** The Login stays the address on the wire, and version 1's "roster rows, blocks, queues and authorization stay keyed by Login" now describes the wire, not a server's storage, so that a relationship can survive a rename.
- **The data-object line corrected.** Version 1 named `0x0628`-`0x062F`; only `0x0628`-`0x062B` were reserved, and `0x062C`-`0x062F` stay the extension's.

## Changes to the Messaging Extension

Each of these reaches only a session confirmed `CAPABILITY_FRIENDSHIPS`, or answers only a request carrying one of version 2's fields:

- **An upgrade offer** is answered `OK`, not `AlreadyFriends` ([Making One](#making-one)).
- **An offer lapses** at its expiry, dropping the offer and keeping the request; the extension's requests never lapse.
- **Remove Friend (803)** may name its party by `DATA_FRIEND_IDENTITY` in place of the extension's REQUIRED `DATA_FRIEND_LOGIN` ([Hidden links](#hidden-links)), and may carry an offer, canceling it alone.
- **Remove Friend (803) and Block User (806)** may carry a withdrawal, and a `Removed` 801 carries one, where the extension's carries only `DATA_FRIEND_LOGIN` ([Showing One](#showing-one)).
- **`AccountNotFound` for an account that exists** and holds `AccessMessaging`, where its link is hidden or does not match the offer, so that an offer or a presentation cannot test a guess at a hidden link.
- **`RosterFull` for a bound that is not the roster**: the presenter's bound on inactive presentations ([Presenting One](#presenting-one)).

---

## Portable Identity

### What Does Not Change

- **The bare identity is still the Login on the wire.** Every transaction names a Login, and roster rows, blocks, queues and authorization are reported under it. The fingerprint travels beside it, never in its place. A server MAY key its own records of a relationship by fingerprint where the account has one, so that the relationship survives a rename; that is invisible on the wire.
- **The [reserved logins](Capabilities-Messaging.md#reserved-logins) and discovery rules apply unchanged**, to the fingerprint as to the Login. A session that proved a key but is logged in as `guest` is still `guest`, and never enters the system. An operator who wants identity users to message gives them accounts of their own.
- **The sender is the session.** The server attributes a message to the Login, and the identity, that the session authenticated as.

### Capability Bits

| Bit | Mask | Name | Meaning |
|---|---|---|---|
| 13 | `0x2000` | `CAPABILITY_IDENTITY` | The client understands the identity field and signed and sealed messages |
| TBD | TBD | `CAPABILITY_FRIENDSHIPS` | The client implements [portable friendships](#portable-friendships) |

A client sets bit 13 when it understands the fields of version 1; it need not have an identity of its own. A server confirms it only when it implements at least version 1 of this document, and only alongside `CAPABILITY_MESSAGING` (bit 6). A client sets `CAPABILITY_FRIENDSHIPS` only beside bit 13, and a server confirms it only beside both. A server sends `DATA_FRIEND_OFFER`, `DATA_FRIENDSHIP`, `DATA_FRIEND_WITHDRAW`, `DATA_FRIENDSHIP_HASH` and `DATA_IDENTITY_ROTATION` only to sessions confirmed both bits; any other session of the same account is sent what version 1 sends, so its Friend Request (804) carries no offer.

A client that sets bit 13 on a session that proved a device key whose certificate allows messages and is at most 512 bytes ([Proving a Device](#proving-a-device)) MUST be able to open sealed messages for that key. Such a session is **sealed-capable**. That is the cost of one bit: a client with such a key that cannot open sealed messages, such as a classic client tunneled under a key, leaves bit 13 off, and so also goes without renames in place and signed messages. Whether a friend's session is sealed-capable needs no bit of its own: [Get User Info (825)](Capabilities-Messaging.md#get-user-info-825) lists a device's certificate only while it has one ([Device Certificates](#device-certificates)).

### The Fingerprint

An account's **fingerprint** is the SHA-256 digest of its 32-byte identity public key ([identity spec section 3.2](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#32-fingerprint)), carried in one new field:

| Field | Notes |
|---|---|
| `DATA_FRIEND_IDENTITY` | The fingerprint of the identity linked to the account, 32 bytes |

- **A server sends it only for an account linked to an identity**, and only where it would show that link to an ordinary user of the server, by its own policy or the account holder's setting, or to a friend the account holder has presented a friendship with ([Hidden links](#hidden-links)). A fingerprint lets anyone recognize the same person on every server where they use the key, so it is the user's to share.
- **Never for an account the server refuses for messaging**: one without `AccessMessaging`, a reserved login, or any other account it treats the same way. Such an account appears nowhere in messaging, and its fingerprint does not either.
- **It is a public-card field**, like `FieldUserName`: sent on entries in every roster state ([Get Roster (800)](Capabilities-Messaging.md#get-roster-800), [Roster Entry (801)](Capabilities-Messaging.md#roster-entry-801)), in [Friend Request (804)](Capabilities-Messaging.md#friend-request-804) and [Presence Changed (809)](Capabilities-Messaging.md#presence-changed-809), and in the results of [Find User (822)](Capabilities-Messaging.md#find-user-822), [User Search (823)](Capabilities-Messaging.md#user-search-823) and [Get User Info (825)](Capabilities-Messaging.md#get-user-info-825), to friends and non-friends alike. It is the hash of a public key and says nothing about whether anyone is signed in.
- It follows the [complete-entry rule](Capabilities-Messaging.md#an-entry-group-is-complete-not-incremental): an entry without it is one whose account shows no identity.
- A server MAY send it to a session that did not negotiate bit 13, which ignores it as an unknown field. Of the rules for receiving it, only the rename rule below depends on the bit.

A client treats the fingerprint as the durable key for a row: two entries with the same fingerprint are the same person, even under different Logins or on different servers, and a client MAY offer to merge local state (aliases, history) across them. The Login remains the address.

### Renames and Rotation

**Rename.** When an account linked to an identity is renamed, a friend's session that negotiated bit 13 receives one Roster Entry (801) for the new Login, carrying `DATA_FRIEND_IDENTITY`, with no `Removed` before it, and MUST rename the row it holds under that fingerprint in place, keeping its alias and local state. Every other session, and every session for an account that shows no identity, receives the removal followed by the fresh entry, as under [Persistence](Capabilities-Messaging.md#persistence). A client that keys rows by Login therefore never sees a Login change without a `Removed`.

**Rotation.** An identity can rotate to a successor key ([identity spec section 8.5](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#85-rotation)), after which the account carries a new fingerprint under the same Login. The server sends a Roster Entry (801) with the new fingerprint to every online friend, and no `Removed`; the account's queue, blocks and receipts follow it unchanged. A client that receives an entry whose Login matches a row it holds and whose fingerprint differs, with no `Removed` for that Login in between, treats it as a rotation: the row keeps its state and takes the new fingerprint. A Login that passes to another account always arrives after a `Removed`, so the two cannot be confused. A client SHOULD tell its user when a friend's fingerprint changes, since on the wire a rotation looks the same as a server substituting a key. A client that implements portable friendships can check the rotation for itself ([Rotation of a Friend](#rotation-of-a-friend)).

### Lookup by Fingerprint

[Find User (822)](Capabilities-Messaging.md#find-user-822) accepts `DATA_FRIEND_IDENTITY` in place of `DATA_FRIEND_LOGIN`. The reply keeps its shape and always carries `DATA_FRIEND_LOGIN`, which the caller addresses from then on. A request carrying both, or neither, is a failure with no reason code.

- **It never answers `NotMessagingEnabled`.** A fingerprint with no account here, linked to an account the server refuses for messaging, or not shown under the rules above, is `AccountNotFound`, as is a subject that has blocked the caller. A fingerprint is the same on every server, so anyone who knows a person's key could otherwise ask each server which Login it belongs to, and pair a known person with an administrative login.
- The rate limits of Find User apply unchanged, and a lookup by fingerprint counts against them as a lookup by Login does.

[User Search (823)](Capabilities-Messaging.md#user-search-823) does not search fingerprints: nobody types a fragment of a hash.

### Proving a Device

A session **proves a device** by signing a fresh challenge from the server with the device key: at login over hxd-ng's [identity spec](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#5-authentication), by a classic session tunneled under such a login, or on any TRTP connection with the [Device Proof extension](Capabilities-Device-Proof.md). A session proved by any of these is proved for this draft, with that device's certificate for the session's life, and with bit 13 is sealed-capable from then on. A client that makes friendship acts proves before Get Roster (800), so that what it sends next is a device act ([Device Acts](#device-acts)).

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

- The sender's session MUST have negotiated bit 13 and `CAPABILITY_TEXT_ENCODING`, and proved a device key. The signature covers the body as UTF-8, with each CR LF pair and then each lone CR replaced by LF, at most 4096 bytes; a client verifies after doing the same to the body it receives.
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

To seal a message, a sender needs the recipient device's certificate. Get User Info (825) returns, to an accepted friend only, one `DATA_FRIEND_DEVICE_CERT` for each of the subject's devices that has a live sealed-capable session: the certificate the device proved ([identity spec section 3.3](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#33-device-certificate)), unexpired, not revoked as far as the server knows, allowing messages, and at most 512 bytes. That a certificate is listed is how a sender knows the device can receive sealed messages now. It is friend-only for the same reason as `DATA_FRIEND_CAPABILITIES`: it changes with the subject's devices and sessions, which a stranger has no business knowing. And, as `DATA_FRIEND_CAPABILITIES` is, the list is withheld from everyone while the subject is `Invisible`: a listed certificate means a live session, so a friend would otherwise learn that an `Invisible` user is online.

**The sending client checks each certificate** against the subject's `DATA_FRIEND_IDENTITY`: that its identity key hashes to the fingerprint, that the identity key's signature on it verifies, that it has not expired, and that it allows messages. It seals only to a certificate that passes, and treats one that fails as absent. A server that handed out bare keys could substitute its own and read everything; a certificate it cannot forge, checked against the fingerprint, is what prevents that.

**The receiving client checks the sender**, in the order the construction gives: the recipient fingerprint, the message's age, that its ID has not been accepted from that sender before, the certificate inside the message against the sender's `DATA_FRIEND_IDENTITY`, and the signature, or for a deniable box that the certificate's `device_enc` key is the one that authenticated it. A message that fails is shown as unverified, never as from that friend.

---

## Portable Friendships

The extension makes a friend an account Login on one server, so a person who uses three servers makes the same friend three times, trusting each server's word each time that the friend is who it says. A **portable friendship** moves the friendship to the two people: it is a pair of objects signed by their devices, naming both identities, which each client keeps and brings to the servers it chooses. The first friend request still trusts the server that carried it to show the right fingerprint; after that the key is pinned in the friendship, and someone on another server under another key is visibly not that friend.

A friendship proves a relationship between two keys, not that a session is the friend: which Login holds a key is a server's word ([Security Considerations](#security-considerations)).

Accounts without an identity, and accounts none of whose clients set `CAPABILITY_FRIENDSHIPS`, keep the extension's model exactly. A client that does not set it, on an account whose other client does, sees the extension's transactions and states, though a friend may appear or leave because of what that other client did. The two kinds of friendship sit side by side on one roster.

### The Objects

Four signed objects, each deterministic CBOR under a domain of its own, signed by a device key and carrying that device's certificate, are defined in hxd-ng's [friendship objects](https://github.com/mishan/hxd-ng/blob/main/docs/proposals/friendship-objects.md) document with their checks; test vectors are to follow there:

- an **offer** (`hl-identity/friend-offer/v1`, at most 1 KiB), from one identity to another's fingerprint, with a random ID and an expiry no more than 30 days after it was signed;
- an **acceptance** (`hl-identity/friend-accept/v1`, at most 1 KiB), by the offered identity, naming the offer by its hash;
- a **friendship**: the offer and its acceptance, as a CBOR array of exactly two byte strings, at most 2080 bytes, named by its **friendship hash**, the SHA-256 of the array;
- a **withdrawal** (`hl-identity/friend-withdraw/v1`, at most 1 KiB), which names a pair of identities by a **pair hash** and ends every friendship between them whose half by one named party was signed before a cutoff, wherever it is presented, for good.

A device's certificate MUST allow messages and be at most 512 bytes, as for a signed message. Times are checked with a skew allowance `S` of ten minutes. Making friends again after a withdrawal takes a half by the party whose side it measured, signed after it: a new offer, or, where that party is the offeree, a new acceptance, even of an offer still open.

### Edges and Consent

An **edge** is a server's friendship between two of its accounts, at most one per pair, of one of two kinds:

- **local**: made by [Add Friend (802)](Capabilities-Messaging.md#add-friend-802) and [Friend Response (805)](Capabilities-Messaging.md#friend-response-805) when the two accounts are not both identity-linked, or a client on either side does not implement this. The extension's friendship, unchanged.
- **portable**: between two identity-linked accounts, **active** once each party has presented a friendship between the two identities on this server. They need not have presented the same one. A local edge between two identity-linked accounts becomes portable when they make a friendship on it, or once both have presented one, keeping its aliases and its queue; until then it stays the local edge it was. A withdrawal presented here ends a local edge between the pair made before its cutoff.

Everything the extension says about an accepted entry applies to both kinds alike. They differ only in how the edge comes to exist and how it ends.

**Each party's consent is per server.** A portable edge is active only once *each* party has presented on that server, so one party cannot make the friendship visible on a server the other has not taken it to. Removing a friend from a device that implements this is a withdrawal, and ends the friendship everywhere it reaches; removing one from a password session, or from a client that sends no withdrawal, ends it on that server only.

### Device Acts

**Offering, accepting, presenting and withdrawing are device acts**: open only to a session that proved a device certificate of the account's linked identity ([Proving a Device](#proving-a-device)), whose certificate is unexpired and not revoked as far as the server knows at the time of the act. An offer, an acceptance or a withdrawal the session gives as its own act is signed by that same certificate. A password session never qualifies, even to an identity-linked account; it can still remove a friend, as the extension requires, but on that server only. This is what bounds a stolen device ([Stolen and Retired Devices](#stolen-and-retired-devices)).

### Making One

A friendship is made when both accounts are identity-linked and the sending client puts an offer, `DATA_FRIEND_OFFER`, in Add Friend (802). An 802 without one makes a local edge.

- **Add Friend (802)** with an offer, from a device-proved session signing with its own certificate. A target that has blocked the sender is answered as the extension answers it, before anything else. The server then checks that the offer's sender is the identity linked to the sending account and its target the fingerprint linked to the target Login, that its signed time is within `S` of the server's clock either way (else `Stale`), and the offer itself. A target that is not linked, whose link the server would not show the sender, or whose fingerprint is not the offer's target, is answered `AccountNotFound`, one answer for all three, so that an offer cannot test a guess at a hidden link. After those, an offer that a withdrawal the server holds would end is refused `Withdrawn`, and the client tries again later. The server stores the offer as it stores a pending request and delivers it in **Friend Request (804)**, beside `DATA_FRIEND_IDENTITY`. An offer past its expiry lapses: the server drops the offer and keeps the request, which a plain 805 then accepts as a local edge.
- **An offer hands its target the sender's device certificate**, which this document otherwise gives friends alone ([Device Certificates](#device-certificates)): its device keys, and its name if it has one. A sender who would rather not name the device to a stranger signs offers from a certificate without one.
- **An upgrade offer**: an 802 with an offer on an existing edge between two identity-linked accounts, local or portable, is not `AlreadyFriends`. It is answered `OK`, the edge stays `Accepted` on both rosters, and the offer is delivered as an 804 only to the target's sessions that negotiated `CAPABILITY_FRIENDSHIPS`, since a client that did not would show a request from someone already its friend. Its acceptance makes a local edge portable, or replaces both parties' presentations on a portable one, keeping the edge either way. Its rejection, or a plain 805 from a client that cannot sign, drops the offer and nothing else. It asks nothing new of the target, so a server's bounds on friend requests do not apply to it. Remove Friend (803) carrying the offer cancels the offer alone.
- **Friend Response (805)** accepting carries `DATA_FRIENDSHIP`, the whole friendship, which the accepting client builds from the offer it was sent, from a device-proved session signing the acceptance with its own certificate. The server checks that it is the stored offer's, that the acceptance's signed time is within `S` of the server's clock either way (else `Stale`), that the acceptance is by the identity linked to the accepting account, and the friendship itself, refused `Withdrawn` if a withdrawal it holds ends it. It stores the edge as portable and as presented by both parties, since they made it here, and sends both parties' sessions the Roster Entry (801) the extension requires, with the friendship's hash in it.
- **A plain 805** accepting an offer with no edge behind it, from a client that cannot sign, makes a local edge; the offer is dropped.
- **Remove Friend (803) on a pending request** cancels it and carries no withdrawal, which would end every friendship between the pair.
- **Crossing requests.** An 802 carrying an offer that meets the target's pending request to the sender is answered `RequestPending`, as the extension has it; one without an offer is answered as the extension, or [Message Requests](Capabilities-Messaging-Requests.md) where a server implements it, answers it; a client that implements this answers the pending request with an 805 instead. Once both accounts of a local edge are linked, whichever client learns it first sends an upgrade offer; two that cross are harmless, since the first acceptance drops the other.

### Showing One

A portable edge's roster entries (800, 801) carry `DATA_FRIENDSHIP_HASH`, the hash of each friendship the parties presented, one or two, to the parties' sessions that negotiated `CAPABILITY_FRIENDSHIPS`, so that each device of each party learns of them once it shares a server with the edge. The friendships themselves are not in the entry: a roster of hundreds sent whole at every login would be megabytes for every device. A client asks for the ones whose hash it does not hold with **Get Friendships**:

- The request carries up to 32 `DATA_FRIENDSHIP_HASH` and `DATA_FRIEND_IDENTITY` fields, in any order.
- The reply's groups are delimited by the field they answer. A hash is followed by its `DATA_FRIENDSHIP`, but only where the caller's account was sent that hash in the entry of an active edge, so that a hash tells a stranger nothing and an inactive edge stays shown to nobody. A fingerprint is followed by the `DATA_IDENTITY_ROTATION` records behind it ([Rotation of a Friend](#rotation-of-a-friend)), only for a fingerprint in an active edge's entry the caller's account was sent.
- It is open to the sessions Present Friendships is, and rate-limited as it is.

A Get Roster whose reply would pass the largest transaction the server accepts streams its entries as separate 801s, one of the forms the extension allows.

A `Removed` entry for a portable edge ended by a withdrawal carries that withdrawal, `DATA_FRIEND_WITHDRAW`, to the same sessions. The extension says a removal carries only `DATA_FRIEND_LOGIN`; this adds a field a client that does not know it ignores.

### Presenting One

**Present Friendships** carries friendships (`DATA_FRIENDSHIP`) and withdrawals (`DATA_FRIEND_WITHDRAW`) in any order, at most 32 objects and at most one withdrawal per pair, since the reply names a withdrawal by its pair hash; a request with more is a failure with no reason code. It is open only to device-proved sessions confirmed bit 6, bit 13 and `CAPABILITY_FRIENDSHIPS`. The server applies a request's withdrawals before its friendships.

A client finds a friend first with Find User (822) by `DATA_FRIEND_IDENTITY`, and presents only friendships whose other party it found, since a fingerprint tells the server less than a friendship, which names both people. A client whose own link is hidden presents without the lookup, since its friend cannot find it until it has.

For each friendship the server checks, in this order:

1. the friendship's objects and signatures, and that one of its identities is the presenting account's linked fingerprint, else `BadSignature`; then, unless the edge is already active, the presenter's own bound on inactive presentations, else `RosterFull`, before anything of the other party is looked at;
2. the other party: no account here, one messaging refuses, or one that has blocked the presenter, or that the presenter has blocked, is `AccountNotFound`, keeping nothing new and changing nothing it already holds, as Find User answers all of them;
3. withdrawals: if one it holds ends the friendship, `Withdrawn`, unless the other party's link is one step 4 would hide, when it is `AccountNotFound`;
4. if the other party's link is one the server would not show the presenter, `AccountNotFound` too, but it records the presentation, since activation needs both parties;
5. otherwise `OK`, and it records that this party has presented it, replacing any friendship it presented for the pair before; a replacement on an active edge sends both parties' sessions its entry again.

When each party has presented a friendship for the pair, the edge is active and both are sent its 801 as for a new friend; activation ends any pending friend request between them. An active edge counts toward each party's `MaxRosterSize`. Past the presenter's own, the presentation is kept and the presenter answered `RosterFull`. Past the other party's, the presenter is answered `OK`, since the other party's roster is not its business, and the edge becomes active when the other party has room. An inactive presentation is shown to nobody and does not count toward the cap; inactive presentations are bounded on their own, at `MaxRosterSize` per presenting account, and past it a presentation is refused `RosterFull` at step 1. Step 2's refusals count toward this bound as recorded presentations do, so that it fills alike whether the other party is absent or hidden.

Step 4 records a presentation and steps 2 and 3 do not, so a reply's timing could tell a hidden party from an absent one; a server makes the two cost alike, or answers both after the same delay.

**The reply** carries a group for each object, delimited by `DATA_FRIENDSHIP_HASH` (the friendship hash, or for a withdrawal its pair hash), each followed by its `DATA_REASON_CODE` and, for `Withdrawn`, the withdrawal where there is one.

| Reason | For | The client |
|---|---|---|
| `OK` (0) | presented, or a withdrawal taken | records it as presented on this server |
| `AccountNotFound` (1) | steps 2 to 4 | tries again at a later login |
| `RateLimited` (10) | past the rate below | backs off |
| `RosterFull` (12) | step 1's bound, or activation past a cap | tries again when it has room |
| `BadSignature` (16) | step 1, or a withdrawal that does not verify or is not the presenter's | drops the object |
| `Withdrawn` | step 3 | with the withdrawal beside it, treats the friendship as withdrawn once it has checked it; with none, stops presenting it on this server |
| `Stale` | a withdrawal's signed time not within `S` of the server's clock | signs it afresh, or tells the user its clock is off |

A client signs only by its own clock, never one a server offers: a server whose clock ran ahead would otherwise collect halves dated in the future, which outlive later withdrawals.

**Order at login.** A client sends its withdrawals before Get Roster (800), so that each is applied before the roster is read; its friendships may follow it.

**Rate.** Each friendship costs several signature checks. Presenting draws on the same per-account rate as Find User, which the extension requires to be rate-limited, one object at a time, and past it answers `RateLimited` rather than spending spam points, so a user who brings hundreds of friends to a new server is paced, not kicked. A server keeps the hash with each presentation, and the same account presenting again under it skips step 1's signatures, though never steps 2 to 4.

#### Hidden links

An account whose identity link the server would not show is not found by others. Presenting a friendship is that account holder's consent to be found by the friend it names, and only by them, until it is withdrawn: Find User, Add Friend and Present Friendships from that friend's account answer as for a shown link, and the friend's entry for it carries `DATA_FRIEND_IDENTITY`. Remove Friend (803) naming a party with which the presenter has only an inactive presentation drops that presentation, so the consent can be taken back on one server, and is answered `OK` whether or not it held one, so that the answer says nothing of a hidden party; it MAY name the party by `DATA_FRIEND_IDENTITY`, since a hidden party's Login is one the presenter cannot learn.

### Withdrawals

**Taking one.** A server takes a withdrawal only fresh, from a device-proved session of its signer, signed by that session's own certificate, with a signed time within `S` of its clock, else `Stale`; nobody presents someone else's. It applies it to whatever presentations and pending offers it ends, and keeps it in any case, so that what it ends cannot be presented here later. One for a pair of which it holds nothing cannot be checked whole, and is kept unapplied until a friendship for that pair is presented. A server keeps at most `MaxRosterSize` of an account's withdrawals, oldest dropped first and the server's own (below) last.

**Applying one**, however it arrives (Present Friendships, Remove Friend (803), Block User (806)), drops the presentations and pending offers it ends, and the edge follows what is left:

- where each party still has a presentation standing, the edge stays active, and both parties' sessions are sent its entry again;
- where one party has none, the edge goes inactive, both are told `Removed` with the withdrawal, and when it is next active it is a new friendship, which takes no message queued before the withdrawal;
- where neither has one, the edge is deleted as the extension's mutual removal deletes it, and both are told `Removed` with the withdrawal.

**Unlinking and deletion.** A server drops an account's presentations when it unlinks the account's identity or deletes the account, with its edges; a link made again starts from the presentations its clients make then. The account's withdrawals are kept, so that what they end stays ended.

**Removal without a withdrawal.** Remove Friend (803) or Block User (806) on a portable edge from a client that sends no withdrawal, or from a password session, ends the edge on this server, and the server records a withdrawal of its own for the remover, dated by its clock, that applies on this server only, so that the remover's other devices do not bring the edge straight back. An 803 or 806 whose withdrawal is refused `Stale` still removes the friend here, and is answered `Stale` so that the client can sign afresh for its other servers.

**Blocking stays a server's matter**, as the extension has it. A block by an identity user withdraws and then blocks locally. The withdrawal reaches the friend's other servers as removal, never as a block, so the extension's block confidentiality holds.

**Spreading.** A client handed a withdrawal ending one of its friendships, in a `Removed` entry or a reply, checks it whole, and does not spread one a device it knows to be revoked signed. Once it has checked one it signs its own for the same pair and cutoff, measured on the same side of the pair, presents it on every server where it presented any friendship it ends, and drops them. So a removal reaches the servers either party brought the friendship to, and a friendship the two have made again since is untouched.

### Rotation of a Friend

A server that has applied a rotation of an identity takes the successor in the predecessor's place in every check above, and refuses every withdrawal signed by the predecessor, or by any key it has seen rotated away from, since what that key signs after the rotation may be a thief's; the successor's client signs the predecessor's cutoffs afresh as its own, one per pair it holds friendships under, naming the predecessor as the party whose side each measures, with a cutoff no later than the rotation. A server that has not applied the rotation refuses the successor's, so there the old friendship can be withdrawn only by the other party until it does. The friendships themselves then move to the successor's key, so that they stop depending on every server having applied the rotation:

- **The record is fetched from the server.** A friend's entry under a new fingerprint, with no `Removed` before it, is a rotation under [Renames and Rotation](#renames-and-rotation). The client asks Get Friendships for that fingerprint, and a server that has applied the rotation answers with the identity's rotation records ([identity registrar, section 4.6](https://github.com/mishan/hxd-ng/blob/main/docs/identity-registrar.md#46-rotation)), one for each rotation of the chain, in chain order. Asking the server that already holds the edge tells it nothing new.
- **The client checks it itself**: both of the record's signatures, the predecessor's and the successor's, and that the successor is the one the predecessor's card committed to ([identity spec section 3.4](https://github.com/mishan/hxd-ng/blob/main/docs/hotline-ng-identity.md#34-user-card)), or, where the card holds no commitment, that the predecessor's registrar published the rotation. A record it cannot check it ignores.
- **Moving.** The successor's client sends each friend an upgrade offer under the successor at the next server they share, and presents the new friendship wherever it presented the old. A friend's client that has checked the rotation, and holds a friendship with the predecessor, or dropped one, that no withdrawal ends other than one signed by the predecessor or its own spread of one, accepts without asking the user and presents the new friendship wherever it presented the old. It withdraws the old pair on a server only once that server's entry carries the new friendship for both parties, so that the edge stays standing. Any other successor's offer is an ordinary friend request.
- **An identity with no commitment and no registrar** cannot rotate in a way a friend can check, so its friends make friends again by hand.

### Stolen and Retired Devices

A device's revocation does not invalidate what it signed: a device is revoked when it is retired as well as when it is stolen, and a revocation does not say since when the device was not its owner's. What bounds a thief is that every friendship act is a device act. Within the time a stolen device can still log in somewhere, until its revocation reaches that server or its certificate expires, the thief can make friendships in the owner's name, present them where it logs in, and withdraw the owner's friendships. Outside it, the thief can make no friendship act at an honest server.

- **A server drops what a revoked device presented.** It records which device's certificate made each presentation, counting an 805 as both parties' presentations by the devices that signed its halves. When it applies a device's revocation it drops that device's presentations and tells both parties `Removed` without a withdrawal; the edge stays inactive with its id and its queue, which are released when it is active again. The owner's other devices then present again what they hold, so a friendship the owner keeps comes back and one only the thief held does not.
- **A friendship the thief made spreads only through the owner's own clients**, which adopt a friendship another of the identity's devices signed only once the user confirms it, shown as the friend, the device that signed and when, grouped by the signing device. One the user does not recognize is withdrawn.
- **Retiring a device.** A client retires a device only once a device that stays has adopted its friendships, since what only the retired device held is lost wherever the revocation is applied. A retired device is revoked rather than left to expire: an expired certificate that is not revoked can still have signed withdrawals dated inside its old validity.
- **A device that holds the identity key** is the identity: revoking it is not enough, and only rotating the identity recovers from its theft.

---

## Additions to the Tables

Data objects. Version 1's are from `0x061F` and `0x0628`-`0x062B`, as the extension reserved them; `0x062C`-`0x062F` stay the extension's. Version 2's are placeholders, too many for the extension's remaining messaging block, for the extension to place:

| ID (hex) | Dec | Name | Type | Description |
|---|---|---|---|---|
| `0x061F` | 1567 | `DATA_FRIEND_IDENTITY` | Binary (32) | The fingerprint of the identity linked to an account; also in Get Friendships and its reply, and in 803 to name a hidden party |
| `0x0628` | 1576 | `DATA_FRIEND_DEVICE_CERT` | Binary (at most 512) | A device certificate, in a Get User Info reply (**repeated**) |
| `0x0629` | 1577 | `DATA_MESSAGE_SIGNED` | Binary (at most 1020) | A signed message in detached form, in 810 and 811 |
| `0x062A` | 1578 | `DATA_MESSAGE_SEALED` | Binary (at most 17,152) | A sealed message, in 810 and 811 |
| `0x062B` | 1579 | `DATA_MESSAGE_SEALED_TO` | Binary (32) | The SHA-256 of the `device_enc` key a sealed message was sealed to, in 810 and 811 |
| TBD | | `DATA_FRIEND_OFFER` | Binary (at most 1024) | An offer, in 802 and 804, and in 803 canceling an upgrade offer |
| TBD | | `DATA_FRIENDSHIP` | Binary (at most 2080) | A friendship, in 805, Present Friendships and Get Friendships' reply |
| TBD | | `DATA_FRIEND_WITHDRAW` | Binary (at most 1024) | A withdrawal, in 803, 806, a `Removed` 801, and Present Friendships and its reply |
| TBD | | `DATA_FRIENDSHIP_HASH` | Binary (32) | A friendship hash, in 800 and 801 (up to two in an entry), Get Friendships and its reply; or a pair hash, in Present Friendships' reply |
| TBD | | `DATA_IDENTITY_ROTATION` | Binary (at most 512) | A rotation record, in Get Friendships' reply. **Repeated**, in chain order |

Every field is well inside the 65,535 bytes a field's length allows, and Present Friendships' largest request, 32 objects of at most 2080 bytes, is about 66 KiB; a client sends fewer objects to a server whose largest transaction is smaller.

Transactions, request/reply, from the extension's unallocated 829-839:

| ID | Name | Request | Reply |
|---|---|---|---|
| TBD | Present Friendships | `DATA_FRIENDSHIP` and `DATA_FRIEND_WITHDRAW`, up to 32 | groups of `DATA_FRIENDSHIP_HASH`, `DATA_REASON_CODE`, and `DATA_FRIEND_WITHDRAW` for `Withdrawn` |
| TBD | Get Friendships | `DATA_FRIENDSHIP_HASH` and `DATA_FRIEND_IDENTITY`, up to 32 | groups of the field answered, then `DATA_FRIENDSHIP` or `DATA_IDENTITY_ROTATION` where there are any |

Capability bit: `CAPABILITY_FRIENDSHIPS`, for the extension to choose ([Capability Bits](#capability-bits)).

Reason codes:

| Code | Name | Meaning |
|---|---|---|
| 15 | `NoDeviceKeys` | A sealed message names a device key no live sealed-capable session of the recipient proved, or the recipient has none |
| 16 | `BadSignature` | A signed message does not verify, is not scoped to the recipient, or reuses the message ID of another message by the same signer; a friendship object that fails its checks |
| TBD (proposed 18) | `Withdrawn` | A withdrawal this server holds ends the friendship |
| TBD (proposed 19) | `Stale` | A friendship object's signed time is not within `S` of the server's clock |

The user keys draft has reasons named `NoDeviceKeys` and `BadSignature` in the link's list, as 13 and 14. They are different numbers because 13 and 14 are taken here. I propose 18 and 19 for `Withdrawn` and `Stale`, after the message requests draft's `Requested` (17), or 17 and 18 if `Requested` is not assigned.

`MessageTooLong` (13) also covers a sealed message over 17,152 bytes, and IM Send's reply may carry `NoDeviceKeys`, `OfflineUndeliverable` or `BadSignature`.

## Security Considerations

- **Accounts without passwords.** An account linked to an identity may log in by proving its key, with no password at all. For such an account the Login being public exposes nothing. The reserved logins rule is unaffected: it is about what those accounts are, not how they authenticate.
- **Fingerprints link people across servers.** Anyone who sees a user's fingerprint can recognize them wherever they use the key, which is why a server sends it only where it would show it to its own users, or to a friend the user presented a friendship with, and never for an account messaging refuses.
- **Sealed messages.** No server can read a sealed message, including the one that carries it. The server still sees who messages whom, when, and roughly how much. A client that seals to a certificate it did not check against the recipient's fingerprint has sealed to whoever supplied the certificate, and the fingerprint is only as good as the server's word: a server that replaces a friend's fingerprint and serves a matching certificate can read what is sealed to it. **On first contact, only comparing fingerprints out of band protects against that.**
- **Signed messages are evidence.** A signature is transferable: whoever holds a signed message, sealed or not, can prove to anyone that the sender's device wrote it. That is what a sender chooses by signing, and why a deniable form exists beside it. Clients SHOULD let the user choose, and SHOULD show which form each message arrived in.
- **Downgrade.** A server can strip a signature or refuse to deliver a sealed message, and a client that silently falls back sends in the clear. This is what protects users, not the server's check. Clients SHOULD remember which friends they have exchanged signed or sealed messages with, and warn before sending to one of them in the clear or when a message from one arrives without its signature.
- **Identifier spoofing.** A server takes `DATA_FRIEND_IDENTITY` from the account it resolved, never from the client, as it does the Login, and a fingerprint in a request is a lookup key, not a claim.
- **What a dishonest server can still do with friendships.** It decides who is sent presence and messages on its own wire, as it does today. It can activate any friendship it has seen, on itself, whether or not both parties presented it there; a client notices only by finding an entry for a friendship it never presented to that server. It can put a friend's genuine friendship on a Login that is not the friend's, since the friendship names keys and the Login is its word; a client that attributes only signed or sealed messages to a friend is not fooled. It can ignore a presentation or a withdrawal, so a removal reaches it only if it chooses to apply it. It can hand a client a withdrawal a stolen device signed, which a client that knows the device revoked does not spread. Friendships constrain what clients believe, not what a server sends.
- **The friend graph.** A client's lookups tell a server which fingerprints it is looking for, and its presentations which friendships it has with those found. A withdrawal's pair hash can be matched by anyone who knows both fingerprints, and servers that compare notes see the same friendship hash at each. That is why presenting is opt-in, per server and per friend.

## Client Behaviour

No client change is required: a client that sets neither bit sees what it sees today, and is never sent a signed or sealed message, an offer or a friendship it cannot read. A client that supports this document:

1. Treats `DATA_FRIEND_IDENTITY`, where present, as the durable key for a roster row, and applies [Renames and Rotation](#renames-and-rotation).
2. Seals only to device certificates it checked against the recipient's fingerprint, and does not fall back to the clear on `NoDeviceKeys`.
3. Shows a signed or sealed message that fails its checks as unverified, never as from that friend, and follows the advice under Downgrade above.

A client that also sets `CAPABILITY_FRIENDSHIPS`:

4. Shows as portable friends only friendships it checked itself.
5. **Presents only where its user chose**, per server and per friend, and asks on a new device before presenting anywhere, since choices made on another device do not reach it. It records which friendships it presented on each server, and presents one again when its edge, once seen active, goes missing from Get Roster. A `Removed` without a withdrawal clears its record for that server.
6. **Adopts** a friendship whose own half this device signed: an offer that comes back accepted, if it first sees the acceptance before the offer's expiry plus `S` by its own clock, or an acceptance it signed. It adopts one another of its identity's devices signed only once the user confirms it, and never one a withdrawal it holds ends.
7. Signs no new half for a pair it holds a withdrawal for until its own clock is past the withdrawal's cutoff, so that a friendship made again is not ended by the withdrawal still spreading.
8. Spreads a withdrawal it has checked, as [Spreading](#withdrawals) describes, waiting where its cutoff is later than its own clock until it is not, and drops every friendship it holds for a pair its own user withdrew from.
9. Prefers a portable friend's user card name, which is the same on every server, over a server's published name, and keeps a local alias for the identity, not per Login.

## Implementation Notes

- hxd-ng implements the identity spec's objects in `crates/hl-identity`, including device certificates and their verification, and will be the reference implementation of version 2. A server that implements only version 1 need check nothing cryptographic: the signature check is a MAY, and only clients open sealed messages. A server that implements portable friendships checks every object it is given, using the checks hxd-ng's [friendship objects](https://github.com/mishan/hxd-ng/blob/main/docs/proposals/friendship-objects.md) define, with test vectors to follow there.
- The construction of signed and sealed messages is the one hxd-ng's [end-to-end messages document](https://github.com/mishan/hxd-ng/blob/main/docs/e2e-messages.md) defines for Hotline-ng and server links. One difference: on a link, a detached signed message carries its certificate's SHA-256 in place of the certificate, to fit Link Chat (904), while here it carries the certificate as that document does, so a message read from the offline queue still verifies after its sender has left.

---

## Possible Follow-ups

**Handles.** A registrar's attestation that a key holds a handle, such as `bob@hl.example`, so that people can find each other by something more readable than a fingerprint. Given on the server's word alone it would be a claim nobody can check, so it waits for the same follow-up as in the [user keys draft](Server-Link-User-Keys.md#possible-follow-ups): an attestation carried with it, and a list of registrars the receiving client trusts. Find User (822) and User Search (823) would then accept it as they accept a Login.

**Friends on linked servers.** Ghosts never take part in messaging, so friendships here are between accounts on one server, each server's edge made from the friendship the two people brought it. A link that carries a ghost's fingerprint lets a client recognize a friend in a linked server's user list, which is not messenger presence. Telling a peer that a ghost is a friend of one of its users, where both presented the friendship at that peer, would let friends see each other's presence across a link, and carrying signed and sealed messages to them needs an exception to the rule that no messaging transaction names a ghost. Both are left for a later version, as an addition to the Server Linking Extension.

**Sealed messages for offline devices.** Sealing mail for a device that is offline needs a construction that marks a message as mail, so the recipient judges it by when it was queued rather than refusing it as stale, and offline queue rules for a sealed body.

**A channel between a person's devices.** Several limits of portable friendships (devices that share no server, choices made on one device reaching another, learning of a revocation without a registrar, hiding the friend graph from servers) come down to a person's devices having no way to reach each other but Hotline servers.

**Withdrawals beyond servers.** A withdrawal reaches only the servers its parties bring it to. Publishing them more widely would publish who stopped being friends with whom to anyone who can guess both fingerprints, and waits on a design that does not.
