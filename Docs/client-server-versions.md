# Known Identifiers

This document catalogs the identifiers that Hotline client and server implementations send during connection establishment. Its purpose is to serve as a public registry so that developers of new clients and servers can select values that do not collide with existing software.

This document does **not** cover HOPE extension App IDs (`DATA_HOPE_APP_ID` / `DATA_HOPE_APP_STRING`).

---

## Version Numbers

During the Login (107) transaction, clients and servers exchange `DATA_VERSION` (field ID 160) — a 2-byte unsigned integer that identifies the software version. Servers also return their own version number in the login reply.

Developers of new software should choose a version number that does not conflict with values already in use.

### Known Client Versions

| Version | Hex      | Software                 | Notes                                     |
|---------|----------|--------------------------|-------------------------------------------|
| (none)  | —        | Hotline Client 1.2.3     | Field not sent; Official Client           |
| (none)  | —        | Hotline Client 1.5 PR c12| Field not sent; Official Client           |
| (none)  | —        | HTLx                     | Field not sent                            |
| (none)  | —        | Ripcord 1.0b15c10        | Field not sent                            |
| (none)  | —        | Hackline 1.0             | Field not sent                            |
| (none)  | —        | StormBot 1.2.1           | Field not sent (HL Bot)                   |
| (none)  | —        | mBot 1.6                 | Field not sent (HL Bot)                   |
| (none)  | —        | Monica 2.1, 2.2          | Field not sent (Download Utility)         |
| (none)  | —        | Zombie b15               | Field not sent                            |
| (none)  | —        | OpenHLC                  | Field not sent                            |
| (none)  | —        | Fancy R6                 | Field not sent                            |
| (none)  | —        | Frogblast                | Field not sent                            |
| (none)  | —        | VertiClient              | Field not sent                            |
| 15      | `0x000F` | Hotline Client 1.5       | Official Client                           |
| 151     | `0x0097` | Panorama 1.1.1           |                                           |
| 151     | `0x0097` | Heidrun                  |                                           |
| 151     | `0x0097` | Hotline Client 1.5.5     | Official Client                           |
| 151     | `0x0097` | Hotline Client 1.7.2     | Official Client                           |
| 151     | `0x0097` | Hotline Client 1.8       | Official Client                           |
| 151     | `0x0097` | Hotline Client 1.8.1     | Official Client                           |
| 184     | `0x00B8` | Hotline Client 1.8.4     | Official Client                           |
| 184     | `0x00B8` | AniClient                |                                           |
| 184     | `0x00B8` | The DefBot 1.0.2 (HL Bot)|                                           |
| 185     | `0x00B9` | Hotline Client 1.8.5     | Official Client                           |
| 185     | `0x00B9` | Pitbull Pro 2.4.4        |                                           |
| 190     | `0x00BE` | Hotline Client 1.9.2     | Last official client                      |
| 190     | `0x00BE` | Underline 1.9.5          |                                           |
| 190     | `0x00BE` | Hotline Client 1.9.5 GPL | Appears to be a modified client           |
| 197     | `0x00C5` | GLoarbLine 1.9.7         |                                           |
| 198     | `0x00C6` | Hermes                   | Modern client; native UTF-8               |
| 199     | `0x00C7` | Klein                    | Modern client; native UTF-8               |
| 255     | `0x00FF` | Hotline Navigator        | Modern client; native UTF-8               |
| 48640   | `0xBE00` | Obsession                |                                           |
| 8867    | `0x22A3` | XCC                      |                                           |

### Known Server Versions

| Version | Hex      | Software                | Notes                                     |
|---------|----------|-------------------------|-------------------------------------------|
| (none)  | —        | Hotline Server 1.2.3    | Field not returned                        |
| 150     | `0x0096` | Hotline Server 1.5      | Returned in login reply                   |
| 151     | `0x0097` | Hotline Server 1.5.5    | Returned in login reply                   |
| 151     | `0x0097` | Hotline Server 1.7.2    | Returned in login reply                   |
| 184     | `0x00B8` | Hotline Server 1.8.4    | Returned in login reply                   |
| 185     | `0x00B9` | Hotline Server 1.8.5    | Returned in login reply                   |
| 190     | `0x00BE` | Hotline Server 1.9      | Returned in login reply                   |
| 190     | `0x00BE` | Mobius                  | Returned in login reply                   |
| 191     | `0x00BF` | Terra 1.2b1             | Returned in login reply                   |
| 196     | `0x00C4` | FreeShare Server 1.0.2  | Returned in login reply                   |
| 197     | `0x00C5` | GLoarbLine Server 1.9.7 | Returned in login reply                   |
| 200     | `0x00C8` | Janus Server 2.0.1      | Returned in login reply                   |
