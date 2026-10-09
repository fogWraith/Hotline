# Server Linking: whether a linked user's connection is encrypted

A proposed addition to the [Server Linking Extension](https://github.com/fogWraith/Hotline/blob/main/Docs/Protocol/Capabilities-Server-Link.md),
from running hxd-ng against Janus with hx-ng and classic clients on the
same network. Not adopted; the field ID is a proposal for the spec's
editor to assign.

## The problem

A user's group says nothing about how that user is connected to their
home server. Every link is protected (the spec requires it), but the
last hop, from the user's client to their home server, can still be a
plain TCP Hotline connection. A private message to that user is then
readable on the wire between their client and their server, however
well every link along the way is protected.

hx-ng marks local users on unencrypted connections and warns before a
private message to one. For a user of a linked server it can't know,
so every remote user is "unknown", and the ones really on cleartext
can't be singled out. I think users should be able to tell whether a
conversation is leaking through someone's unencrypted connection,
wherever that person is in the network.

## The proposal

A new user group field, set by the user's home server only:

| Field ID | Name | Type | Meaning |
|---|---|---|---|
| `0x0645` | `DATA_LINK_USER_TRANSPORT` | UInt16 | How the user is connected to their home server right now: `1` encrypted, `2` cleartext, `3` weakly encrypted |

- **Encrypted (`1`)** means the connection is protected against
  eavesdropping: TLS terminated by the server itself (including a
  WebSocket over TLS), or a HOPE transport cipher. It doesn't say the
  client verified the server: a home server can't tell whether a
  client verified its certificate, so the field can't promise more than
  "not readable by a passive observer", and I'd rather it not pretend
  to. HOPE's ChaCha20-Poly1305 counts, and so does Blowfish, which
  older clients negotiate. **Cleartext (`2`)** is a plain TCP
  connection, or a session that used HOPE only to log in.
- **Weakly encrypted (`3`)** is a HOPE session whose transport cipher is
  RC4. It isn't cleartext, but it isn't "not readable in passing"
  either: RC4's keystream biases let a passive observer recover
  repeated plaintext, which is why TLS prohibits it (RFC 7465). Many
  legacy HOPE sessions use RC4, so a value of its own lets a client say
  what is true of them rather than round either way. A receiver that
  predates `3` treats it as unknown, never as encrypted.
- **A home server MUST NOT send `1` on an assumption it hasn't
  checked.** One that can't vouch for the connection (behind a proxy
  that terminates TLS, or a plain listener it only assumes is behind
  one) omits the field.
- **Absent means unknown**, never "encrypted". As a group is complete,
  a group without the field clears any earlier value. Receivers treat
  `0` and any value they don't know as unknown too, so a value added
  later can't be mistaken for encrypted by an older receiver, and accept
  the integer in either 2- or 4-byte width.
- **It describes the current connection.** The home server sends a new
  group whenever the value changes, including when a session resumes on
  another connection (hxd-ng's sessions can). It's not a promise about
  delivery: a message held for a user who is away goes out over whatever
  connection they come back on.
- **Relayed unchanged**, like every other group field. Relaying Fields
  already has servers pass on fields they don't know, so a relay that
  predates this field carries it anyway and only home servers have to
  change (Janus does, since 2.0.19). It sits outside the baseline, so it counts toward the 1024-byte
  budget, at six bytes.
- **Why not a flag bit:** each hop rewrites `FieldUserFlags` by its own
  rules and only bits 0-3 are defined, so an older relay may clear a new
  bit; one bit can't say "unknown"; and flag bits are vocabulary classic
  clients see directly.

## Two things the spec says that this touches

- **"Senders MUST NOT send ... connection details."** The rule is about
  identifying details, and a two-value class is not one of them, but
  it's close enough to say so: amend it to "connection details (address,
  port, client software)", and name `DATA_LINK_USER_TRANSPORT` as not
  one of them.
- **Privacy.** The field does tell the network which users are on
  cleartext. I think that's the point: the people they talk to are the
  ones exposed by it, and the user's own server already shows it to its
  own clients today (hxd-ng does, to hx-ng). But it belongs in
  Security Considerations beside "no personal data crosses".

## What it doesn't do

- It's the home server's word, and only that. It doesn't protect against
  an operator running software that lies about it, any more than the
  rest of the extension protects against a server lying about its own
  users. But a user whose server's operator acts in bad faith has bigger
  problems than this field: that server reads their messages anyway.
- "Encrypted" is about the wire. Every server on the path can still
  read a message, as the Trust Model already says; end-to-end encryption
  is the user keys draft's job.

## Settled with fogWraith

- Janus relays user group fields it doesn't know since 2.0.19, copying
  the whole entry in order.
- "Connection details" is amended as above.
- Janus terminates TLS itself, with no proxy listener, and records the
  HOPE cipher per session (ChaCha20-Poly1305, RC4, Blowfish, or none
  when HOPE only logged the user in). A session's encryption never
  changes during its life, so Janus can always send a value and never
  needs to omit the field.
- A value for "encrypted, and the client verified the server" is left
  room for, not defined yet. HOPE's password-derived keys come
  close, but not for a guest account with an empty password, and that
  distinction belongs in its own value later.
- `0x0645` sits in the range the spec reserves for future link growth,
  so it is fogWraith's to assign; the user keys draft skips it for
  that reason.

## On hxd-ng's side

hxd-ng knows this for its classic and TLS ports. For its WebSocket port
it currently assumes the plain listener is behind a TLS proxy, so I'd
have it omit the field there unless the operator vouches for the proxy,
rather than send `1` on that assumption. A `/trtp` tunnel reports its
own hop, which hxd-ng uses. hxd-ng has no HOPE yet, so the HOPE cases
are Janus's for now. On receipt it maps `1` and `2` to hx-ng's
`encrypted` and `cleartext`, and absent stays `unknown`; `3` needs a
state of its own in hx-ng. None of it is
built yet.

Until it's adopted, `0x0645` is a trial field, sent only to peers whose
operators agree to it, as the spec allows for drafts. **A trial doesn't
stay between those two servers**, though: every server behind the peer
relays it, since relays pass on fields they don't know, so which users
are on plain connections becomes visible to servers that never opted in.
An operator agreeing to the trial agrees to that for the network beyond
its peer, and a server that would rather not show it to anyone omits
the field.
