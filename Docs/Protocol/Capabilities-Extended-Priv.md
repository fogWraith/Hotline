# Extended Privilege Bitmap Extension

> Last updated: October 1, 2026

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document describes the extended privilege bitmap extension to the Hotline protocol. It widens `FieldUserAccess` (110) from an 8-byte (64-bit) to a 16-byte (128-bit) bitmap when both peers negotiate support via capability bit 5, giving the access system room to grow past bit 63.

Nothing else changes. The extension defines no permission, no transaction and no field; bit *n* means the same thing at either width. Legacy clients and servers are unaffected - they exchange the 8-byte bitmap exactly as they always have.

For the general capability negotiation mechanism, see [DATA_CAPABILITIES](Capabilities.md).

## Table of Contents

- [Background](#background)
- [Capability Bit](#capability-bit)
- [Wire Format](#wire-format)
- [Affected Transactions](#affected-transactions)
- [Width Selection](#width-selection)
- [Preserving Bits a Client Cannot See](#preserving-bits-a-client-cannot-see)
- [Account Storage](#account-storage)
- [Server Behaviour](#server-behaviour)
- [Client Behaviour](#client-behaviour)
- [Conformance Test Vectors](#conformance-test-vectors)
- [Defining New Privilege Bits](#defining-new-privilege-bits)
- [Security Considerations](#security-considerations)
- [Bit Allocation](#bit-allocation)

---

## Background

`FieldUserAccess` (110) is an 8-byte bitmap. Hotline 1.9 defined bits 0–40, GLoarbLine took 41–54, and five extensions have since taken 55–60 (see [Bit Allocation](#bit-allocation)). **Three bits remain: 61, 62 and 63.**

The count matters less than the rate. Bits 55–60 were allocated inside roughly a year, and a single extension routinely wants more than one - video took two because publishing a camera and publishing a desktop are different permissions. Three bits is one or two more extensions, not three.

The cost lands on whichever extension arrives first to find none left, because that extension cannot ship until the wider bitmap ships across the server *and* every client that edits accounts. That is why this document exists before anything needs it: the width change is invisible while no bit above 63 is defined, so it can be built and deployed quietly, and the first extension to allocate bit 64 finds the path already there.

---

## Capability Bit

| Bit | Mask | Name | Description |
|---|---|---|---|
| 5 | `0x0020` | `CAPABILITY_EXTENDED_PRIV` | Both peers exchange `FieldUserAccess` (110) as a 16-byte (128-bit) bitmap |

The bit is carried in `DATA_CAPABILITIES` (`0x01F0`) during Login (107) and confirmed in the login reply, per [DATA_CAPABILITIES](Capabilities.md). It has no dependency on any other capability bit.

Negotiation is **ungated**: a server that implements this extension confirms bit 5 whenever the client advertises it, and has no reason to withhold it. There is no configuration setting to enable or disable the extension.

A server **MUST NOT** confirm bit 5 unless the client advertised it. [DATA_CAPABILITIES](Capabilities.md) permits servers to grant some capabilities a client did not ask for; that allowance does not apply here, for the same reason it does not apply to `CAPABILITY_MODERN_DATES` - the bit selects a wire format the client must already be able to parse, not a feature.

---

## Wire Format

`FieldUserAccess` (110, `0x006E`) carries the bitmap as a contiguous big-endian byte sequence. The receiver determines the width from the field's length prefix:

| Length | Encoding | Contents |
|---|---|---|
| 8 bytes | Legacy | Bits 0–63. Standard Hotline encoding. |
| 16 bytes | Extended | Bits 0–127. Bytes 0–7 are byte-for-byte identical to the legacy encoding; bytes 8–15 carry bits 64–127. |

Bit numbering is unchanged from the legacy encoding and extends by the same rule:

```
byte index = n / 8
bit mask   = 1 << (7 - (n mod 8))
```

Bit 0 is the most significant bit of byte 0; bit 64 is the most significant bit of byte 8.

**Example.** An account holding `AccessDownloadFile` (2), `AccessReadChat` (9), `AccessSendChat` (10), `AccessMessaging` (58) and a hypothetical bit 70:

```
Extended (16 bytes):
  20 60 00 00 00 00 00 20  02 00 00 00 00 00 00 00
  └────── bits 0-63 ─────┘  └───── bits 64-127 ────┘

Legacy (8 bytes) - same account, sent to a legacy peer:
  20 60 00 00 00 00 00 20     (bit 70 not representable)
```

Byte 0 `0x20` is bit 2; byte 1 `0x60` is bits 9 and 10; byte 7 `0x20` is bit 58; byte 8 `0x02` is bit 70.

**Any other length is malformed.** A receiver MUST NOT zero-pad or truncate it. A server MUST reply with an error transaction and leave the account unmodified; a client MUST NOT present for editing an account whose bitmap it could not parse. A server MUST also reject a 16-byte field from a client that did not negotiate the capability - accepting it would let any client opt itself into the wide encoding without negotiating.

---

## Affected Transactions

| Transaction | ID | Direction | Role of `FieldUserAccess` |
|---|---|---|---|
| List Users (reply) | 348 | S → C | Every account's bitmap, once per account, inside a `FieldData` sub-field list |
| Update User | 349 | C → S | The bitmap for each account in the batch, inside `FieldData` |
| New User | 350 | C → S | The bitmap for an account being created |
| Get User (reply) | 352 | S → C | The named account's bitmap, for an account editor |
| Set User | 353 | C → S | The bitmap for an existing account |
| User Access | 354 | S → C | The session's own bitmap, sent after login and on any change |

A server MUST apply the same width selection to all six, **including any server-specific transaction of its own that carries the field**. Sending 16 bytes in a Get User (352) reply and then accepting only 8 on the matching Set User (353) defines no coherent round trip; sending 16 bytes there and 8 in the List Users (348) reply hands one session two encodings of the same account.

> **Note.** List Users (348) is the path most likely to be missed. It is not in the original Hotline protocol, it carries the bitmap nested inside a sub-field list rather than as a top-level field, and it is commonly built by serialising the stored account record directly - a code path with no session in scope to ask about width. It still MUST follow the session's negotiated width.

`FieldUserAccess` does not appear in Get User Name List (300), Get Client Info Text (303), or the Login (107) reply.

---

## Width Selection

Width is per-session and per-direction, and follows the negotiated capability alone:

| Sender | Peer negotiated bit 5? | Width sent |
|---|---|---|
| Server → client | Yes | 16 bytes |
| Server → client | No | 8 bytes |
| Client → server, editing an account | Yes | The width that account was received as |
| Client → server, creating an account | Yes | 16 bytes |
| Client → server | No | 8 bytes |

1. A capable server MUST send 16 bytes to a capable client and 8 bytes to every other client.
2. A capable client MUST accept both widths, inferring the width from the length prefix.
3. A capable client MUST send 8 bytes to a legacy server, which has no contract to round-trip bits 64–127.
4. A legacy client never receives 16 bytes, because the server downgrades. When it sends 8 bytes to a capable server, bits 64–127 are **absent**, not clear - see below.
5. **Creation has no received width to echo.** New User (350), and the create branch of Update User (349), describe an account the client has never been sent, so a capable client MUST send 16 bytes and a legacy client MUST send 8. Nothing is preserved on a create, so the width carries no information beyond which bits the client is able to state at all.

---

## Preserving Bits a Client Cannot See

Hotline servers already merge rather than overwrite on an account edit, because a client that understands only bits 0–*N* would otherwise wipe everything above *N* when it saves. This extension adds one tier at the top and changes nothing else about the mechanism.

### Tiers

A session's **tier** is the highest bit its account editor can see, and so the highest it may change. Each tier is a superset of the one below it, and is named by that bit.

| Tier | Clients | How the server determines it |
|---|---|---|
| 31 | Hotline 1.2.3 | No `FieldVersion` (160) in the login request |
| 40 | Hotline 1.5 / 1.9, and anything unrecognised | **Default** |
| 54 | GLoarbLine | `FieldVersion` = 197 |
| 63 | Modern clients that keep the full legacy bitmap | Operator configuration listing the client's version |
| 127 | Clients implementing this extension | `CAPABILITY_EXTENDED_PRIV` confirmed for the session |

Tier 63 is the one this extension is most likely to be implemented on top of incorrectly, so it is stated explicitly: a modern client whose account editor round-trips all 64 legacy bits, but which has not implemented this extension, sits at **63 - not at 127, and not at 40**. That property cannot be inferred from a version number (it is a fact about the editor, not about who wrote it), which is why the tier is operator configuration rather than a table.

**An operator's client-version list tops out at 63.** Only a confirmed bit 5 reaches 127. A server whose "full bitmap" constant is raised from 63 to 127 while a configured version list still returns that constant has granted every listed client a tier it cannot honour - the list says the editor keeps bits it does not understand, which is a claim about 64 bits; 127 is a claim about the wire, and only negotiation can make it.

### The merge rule

On every edit of an existing account, the server computes the highest bit the editing session may change:

```
max_bit = tier(session)                 // 31 / 40 / 54 / 63, or 127 if bit 5 was confirmed
if received field is 8 bytes:
    max_bit = min(max_bit, 63)          // an 8-byte field says nothing about 64-127
max_bit = min(max_bit, highest_defined_bit)
```

then merges:

1. Bits `0..max_bit` come from the client, with any bit this server does not define cleared.
2. Bits above `max_bit` come from the **stored** account, unmasked.

Order is normative: merge first, then mask, and mask only the range the client was authoritative over. The preserved bits did not come from the client - masking them clears exactly what the merge exists to protect.

**"Defined" means defined by this server's own code** - a bit it assigns a meaning to and enforces somewhere - not a bit the registry has allocated. A server that has not implemented video treats bits 59 and 60 as undefined even though [Bit Allocation](#bit-allocation) reserves them, and clears them out of an incoming bitmap like any other undefined bit. `highest_defined_bit` is the highest such bit even when the set below it has holes; the holes are handled by the masking step, not by the clamp.

Two of the three clamps are new here:

- **The 8-byte clamp** stops a capable client that sent a legacy-width field from having bits read past the end of it.
- **The defined-bit clamp** matters because a session at 127 would otherwise preserve nothing, leaving the server trusting the client to round-trip all 128 bits. No client can meaningfully edit a bit the server assigns no meaning to, so taking those from storage costs nobody anything. Example: a server defines bits 0–60 and an account carries bit 70, put there by a direct edit to the store or by a record written when the server did define it. A capable client that knows nothing of bit 70 sends it clear; `max_bit` is 60, and bit 70 survives.

> **Note.** Whether that example is reachable at all depends on the store. A server with a **sparse** store cannot hold a bit it has no name for ([Account Storage](#account-storage)), so its above-`max_bit` range is always clear and the clamp is a no-op that costs nothing. On a **dense** store the bits are real and the clamp is what keeps them. Implement it either way: a server can change stores, and a server that defines a bit and later retires the code behind it is left holding exactly this case.

Only a confirmed bit 5 raises a session to 127. Version inference, an operator's client-version list, and any other capability bit MUST NOT - the tier is a claim about wire width, and asserting it for a client that speaks 8 bytes discards bits 64–127 on that client's next save, silently and permanently.

### Account creation

New User (350), and the create branch of Update User (349), have no stored value to merge. The server takes bits `0..max_bit` from the client, clears everything above, clears any bit it does not define, and MAY substitute its configured new-account default if the result is empty. That default MUST be safe with every high bit clear.

The default is the operator's policy, not the client's request, so the [escalation guard](#privilege-escalation-checks) does not apply to it: a client that sent an empty bitmap asked for nothing, and the account gets what the operator configured. A server MUST NOT refuse such a creation because the creating session lacks a bit the default holds.

### Batches

Update User (349) carries a list of accounts, so "reject the transaction and leave the account unmodified" needs to say *which* account. A server MUST validate the width of **every** `FieldUserAccess` in the batch before applying **any** of them, and reject the whole transaction if one is malformed. Applying records up to the bad one and then erroring leaves the operator with a partially applied edit and an error message, and no way to tell which half landed.

### Privilege escalation checks

A server implementing this extension MUST refuse any account edit or creation that grants a privilege the acting session does not itself hold, and MUST make that check over all 128 bits; one that stops at 64 lets a capable client grant bits 64–127 it does not have. The guard applies to edits as much as to creation: without it on the edit path, any session that may modify accounts can grant itself, through a second account, everything it lacks.

The check MUST run over the bits the operation **adds** (`result &^ stored`), not over the whole resulting bitmap. Checking the result breaks the merge: a preserved high bit the editing session does not hold - precisely the bit the merge protects - would make an ordinary operator unable to edit any account a more privileged one had touched. For a creation, `stored` is empty, so every bit the client sent is an added bit.

**Order is normative:** merge, then mask, then guard.

1. **Merge** takes bits `0..max_bit` from the client and the rest from storage.
2. **Mask** clears every bit in `0..max_bit` the server does not define.
3. **Guard** refuses the operation if any bit it adds is one the acting session does not hold.

Masking comes before the guard because a session can never hold an undefined bit: guarding first would refuse an edit that merely carried a bit the server ignores, where the mask's job is to drop that bit and apply the rest. A refused operation leaves the account unmodified, exactly as a rejected width does.

### Administrative paths that are not the Hotline wire

An HTTP admin API, a CLI tool or a direct store edit has no login reply in which to negotiate. Such a surface is part of the server, not a foreign peer, so it operates at `max_bit = 127` (still clamped by the highest defined bit) and MUST route through the same merge-and-mask path. A surface that assembles a bitmap independently - from named booleans, say - produces a full value with unknown bits clear, and writing that is an overwrite wearing the shape of an edit.

Surfaces that name permissions rather than numbering them need no change when the bitmap widens. One that exposes the bitmap as a number does: JSON cannot carry 128 bits exactly, so use a string or an array of names.

> **Widening the bitmap is not complete until every path that can write an account has been widened.** A single missed path truncates the whole map on its next write, with no peer having done anything wrong.

---

## Account Storage

The extension imposes one storage requirement: **an account's stored form must be able to hold a bit above 63.**

- **Dense stores** (a packed bitmap on disk) MUST widen the field to at least 16 bytes, padding existing records with eight zero bytes. Migration is atomic per record; a half-migrated store is fine across a run, since each record carries its own length, but a half-written record is not.
- **Sparse stores** (named permissions, e.g. a YAML mapping of names to booleans) need no migration. Bit indices never appear on disk, so existing account files are already correct at any width, and a bit above 63 becomes storable in the same change that names it - exactly as bits 41–58 were.

On load, absent upper bytes read as zero. On save, every bit the store can represent is preserved.

> **A sparse store silently drops any bit it has no name for.** That is a property of sparse storage at any width, not something this extension introduces, but it interacts with the merge rule and implementers should know which way: such a server can never accumulate a bit above its defined set, because the masking step clears undefined bits on the way in and the store would drop them on the way out regardless. The visible consequence is that a bit which is *allocated but not implemented here* - 59 and 60 on a server without video - does not round-trip. Masking makes that explicit at the merge instead of losing it silently at the next save.

---

## Server Behaviour

A server implementing this extension MUST:

1. Confirm `CAPABILITY_EXTENDED_PRIV` whenever the client advertises it, and never otherwise.
2. Send 16 bytes to capable clients and 8 bytes to everyone else, on all of transactions 348, 349, 350, 352, 353 and 354, and on any server-specific transaction that carries the field.
3. Accept 8 or 16 bytes from a capable client, 8 bytes only from a legacy client, and reject every other length with an error transaction that leaves the account unmodified.
4. Apply the [merge rule](#the-merge-rule) on every edit, through every administrative path.
5. Refuse any edit or creation that adds a bit the acting session does not hold, checking all 128 bits, over added bits only, after merging and masking.
6. Store and reload all 128 bits without loss.

A server carries all of this **before** any bit above 63 is allocated, and whether or not one ever is. Deferring the work means shipping storage, merge, wire and administrative changes in the same release as the feature that needs them - the coordinated release this extension exists to avoid.

A server that has not implemented the extension MUST NOT confirm the capability bit. Its behaviour is otherwise unchanged.

---

## Client Behaviour

A client implementing this extension MUST:

1. Advertise `CAPABILITY_EXTENDED_PRIV` only if it parses, renders and round-trips the 16-byte encoding. Merely tolerating the field is not enough.
2. Accept `FieldUserAccess` of length 8 or 16, inferring the width from the length prefix, and reject any other length rather than padding it. When the field is 8 bytes, treat bits 64–127 as *unknown*, never as clear.
3. Send back the width it received for that account.
4. Edit the bytes the server sent, in place. A client MUST NOT rebuild the bitmap from the set of named permissions it knows about - that clears every bit it does not.
5. Render only the bits it recognises. Unknown bits MUST be hidden or shown as opaque reserved entries, never as cleared toggles a user could believe they are setting.
6. Treat the server, not its own copy, as authoritative after an edit. Set User (353) and Update User (349) reply without a bitmap, and the server may legitimately have cleared bits the client sent - any bit above the session's `max_bit`, and any bit this server does not define. A client that keeps showing what the user ticked is showing something the server did not store. Re-read the account with Get User (352) after a successful edit, or refresh from the next List Users (348).

Clients that do not implement the extension need no changes.

> **Note.** Requirement 6 is not specific to this extension - masking and preservation have always been silent - but the extended range makes the gap wider and the failure ("I set that bit and it didn't save") is reported against the client, not the server.

---

## Conformance Test Vectors

These fixtures are normative. Every implementation SHOULD test against these exact bytes, so that a disagreement between a server and a client is a disagreement about one named vector rather than about two different test suites.

**The fixtures.** All merge vectors use one stored account, one incoming bitmap, and a server that defines bits 0–58 and nothing above:

```
stored    20 60 00 00 00 00 00 20  02 00 00 00 00 00 00 00
          bits 2, 9, 10, 58, 70

incoming  00 60 00 00 80 20 00 00  00 00 00 00 00 00 00 00
          sets bits 9, 10, 32, 42; clears bits 2, 58, 70
          (legacy-width senders send the first 8 bytes)

server    defines bits 0-58; highest_defined_bit = 58
```

### Merge vectors

| # | Editing session | Field width | `max_bit` | Result |
|---|---|---|---|---|
| M1 | Tier 31 (no `FieldVersion`) | 8 | 31 | `00 60 00 00 00 00 00 20` + `02 00 …` |
| M2 | Tier 40 (default) | 8 | 40 | `00 60 00 00 80 00 00 20` + `02 00 …` |
| M3 | Tier 54 (GLoarbLine) | 8 | 54 | `00 60 00 00 80 20 00 20` + `02 00 …` |
| M4 | Tier 63 (operator-listed) | 8 | 58 | `00 60 00 00 80 20 00 00` + `02 00 …` |
| M5 | Tier 127 (bit 5 confirmed) | 16 | 58 | `00 60 00 00 80 20 00 00` + `02 00 …` |
| M6 | Tier 127, sending legacy width | 8 | 58 | `00 60 00 00 80 20 00 00` + `02 00 …` |

Read down the tiers: bit 2 is cleared by everyone, because every tier can see it. Bit 32 is taken from the client only from tier 40 up (M1 preserves it clear from storage). Bit 42 only from tier 54 up. Bit 58 is cleared only from tier 63 up - the tiers that can see it. **Bit 70 survives every row**, including M5 and M6, because the defined-bit clamp holds `max_bit` at 58 no matter how capable the session is.

M5 and M6 producing identical results is the point of the 8-byte clamp: a capable client that sends a legacy-width field must not have bits read past the end of it.

### Rejection vectors

Each MUST produce an error reply with the account **byte-identical** to `stored` afterwards.

| # | Input | Why |
|---|---|---|
| R1 | 16-byte field from a session that did not confirm bit 5 | Opting into the extended tier without negotiating |
| R2 | 12-byte field from any session | Not 8 or 16 |
| R3 | 4-byte field from any session | Not 8 or 16 - a short field MUST NOT be zero-padded |
| R4 | 0-byte field from any session | Not 8 or 16 |
| R5 | Update User (349) batch of three accounts where the second carries a 12-byte field | Whole batch rejected; none of the three modified |

> R3 is the one that catches the most common implementation shortcut. Copying an incoming slice into a fixed-width buffer accepts a 4-byte field and leaves bits 32–63 as whatever the buffer held - zero - which clears privileges no client asked to clear. It is also the vector a hostile client would actually send.

### Masking vectors

| # | Setup | Expected |
|---|---|---|
| K1 | Capable session sets bit 59 on a server defining 0–58 | Bit 59 cleared; rest of the edit applied |
| K2 | Server defines 0–58 and 60, not 59; capable session sets both 59 and 60 | Bit 59 cleared, bit 60 kept - the mask handles holes, the clamp does not |
| K3 | Capable session sets bit 100 on a server defining 0–58 | Bit 100 cleared |

### Creation vectors

| # | Setup | Expected |
|---|---|---|
| C1 | Tier 40 session creates an account, sends 8 bytes with bit 58 set | Bit 58 cleared - creation clamps at `max_bit` too |
| C2 | Capable session creates an account, sends 16 bytes with bit 70 set, server defines 0–58 | Bit 70 cleared; creation preserves nothing |
| C3 | Any session creates an account with an all-zero bitmap, on a server that has a new-account default | The default applied, even where it holds bits the creating session lacks |
| C4 | Capable session creates an account with a non-zero bitmap | Default NOT applied; the client's bits stand |

### Escalation vectors

E1 uses its own server, one that defines bit 70; the other rows use the fixture's server. On the fixture's server bit 70 is undefined, so masking clears it before the guard runs (C2, K3) and there is nothing to refuse.

| # | Setup | Expected |
|---|---|---|
| E1 | Server defines bits 0–58 and 70; capable session lacking bit 70 creates an account with bit 70 set | Refused - the guard iterates 128 bits, not 64 |
| E2 | Session lacking bit 58 edits an account that holds bit 58, from tier 40 | **Allowed.** Bit 58 is preserved, not added, so the guard must not see it |
| E3 | Capable session lacking bit 58 sets bit 58 on an account that lacks it | Refused - the guard covers edits, not only creation |
| E4 | Capable session lacking bit 58 sets bit 59 (undefined) and bit 9 (held) on an account | Allowed: bit 59 masked, bit 9 applied - masking runs before the guard |

E2 is the vector that fails when a guard is written over the resulting bitmap instead of over the added bits, and it fails in the most confusing possible way: an ordinary operator can no longer edit any account a more privileged operator has touched.

### Width vectors

| # | Setup | Expected |
|---|---|---|
| W1 | Capable session sends Get User (352) | Reply carries a 16-byte `FieldUserAccess` |
| W2 | Legacy session sends Get User (352) | Reply carries an 8-byte `FieldUserAccess` |
| W3 | Capable session sends List Users (348) | **Every** nested `FieldUserAccess` is 16 bytes |
| W4 | Legacy session sends List Users (348) | Every nested `FieldUserAccess` is 8 bytes |
| W5 | Account edited by another operator while its owner is connected | The owner's User Access (354) push uses the **owner's** negotiated width, not the editor's |
| W6 | Capable client receives 352, echoes it back unmodified via 353 | Account byte-identical, all 128 bits |
| W7 | Legacy client receives 352 (8 bytes), echoes it back via 353 | Bits 64–127 unchanged in storage |

W5 is a distinct code path from every other row - a server-initiated push rather than a reply - and it is the one place where the width belongs to a session other than the one that caused the transaction.

---

## Defining New Privilege Bits

This document defines no permission, but it makes new ones possible, so it is the right place to constrain them. These rules apply to any bit above a client's tier - they are not specific to bits 64–127, and the problem exists today between GLoarbLine's bits 41–54 and the many deployed tier-40 clients.

**The invariant: clear means "as before".** Every path that cannot see a bit produces that bit clear - a 1.5 client editing an account, an account file written before the bit existed, a template that never mentioned it. So a bit MUST be defined such that clear reproduces the behaviour that existed before the bit was defined, and setting it may only ever add capability.

Three consequences:

- **Express permissions as grants, never as restrictions.** A bit meaning *"may do X"* is safe: clear denies something nobody had. A bit meaning *"may **not** do X"* has its safe state as *set*, and every path that cannot see it produces clear - so an account created through a legacy client is silently the unrestricted one. That is a privilege escalation requiring no attack, only an old client. The classic protocol's apparent counter-examples hold up: `CannotBeDisconnected` and `NoAgreement` are grants of *exemption* - set confers a bypass, clear is what everybody had before.
- **Gate only new functionality.** Never put a new bit in front of something clients already do, because clearing it removes the capability, and clear is what every legacy path produces. Retroactively restricting an always-ungated capability is not expressible in the bitmap; that policy belongs in server-wide configuration, which applies regardless of what a client can see.
- **Never rely on a default of *set*.** Defining a bit for existing functionality and setting it on every current account does not hold: the merge preserves it on edits, but nothing sets it on an account *created* afterwards through a path that does not know the bit. A new-account template MUST be safe with the bit clear.

**A bit is granted only through a path that can represent it** - a client whose tier reaches it, the server's own admin API, or a direct store edit. A path that cannot see a bit has no business changing it, and the merge rule guarantees it will not. This is not a limitation to work around: do not gate a new capability on the new bit *or* an older one within reach to make it reachable, because that grants the capability to every account already holding the older bit, on every deployment, with no per-account review. An extension defining a high bit SHOULD say plainly which tools can grant it.

---

## Security Considerations

- **Privilege loss via downgrade.** A legacy client editing an account it cannot fully see would otherwise clear bits set from a capable client. The merge rule prevents it by construction, and MUST be implemented for tier 127, not only the legacy tiers.
- **Fabricated bits.** A malicious capable client may set bits the server does not define. Bits above `max_bit` are taken from storage; within the authoritative range the server MUST clear any bit it does not define - including gaps, so a server defining 0–58 and 60 but not 59 MUST NOT accept a client's bit 59.
- **Merge and mask are not alternatives.** Masking the preserved range instead of merging it clears exactly the bits the rule exists to protect.
- **Narrow escalation guards.** A guard iterating only 64 bits lets a capable client grant bits 64–127 it does not hold.
- **Capability spoofing.** Rejecting a 16-byte field from a client that did not negotiate is what stops a client granting itself the wide encoding.
- **Store integrity.** A dense store must migrate atomically per record; a crash mid-record leaves a width that no longer matches its content.
- **Defaults.** Migration sets all new bits to zero, which is the safe value. The realistic risk is an operator flipping newly defined bits on in a guest template without review.

---

## Bit Allocation

| Bits | Consumed by |
|---|---|
| 0–40 | Hotline 1.9 |
| 41–54 | GLoarbLine |
| 55 | Voice chat - `AccessVoiceChat` |
| 56 | Chat history - `AccessReadChatHistory` |
| 57 | Inline media - `AccessSendMedia` |
| 58 | Instant messaging - `AccessMessaging` |
| 59–60 | Video chat - `AccessVideoChat`, `AccessScreenShare` |
| 61–63 | **Unallocated.** Treat as reserve, not inventory - see [Background](#background). |
| 64–127 | Available only to sessions that negotiated this capability. No bit allocated. |
