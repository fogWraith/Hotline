# Software cards

> Part of the [card authors' guide](README.md). Card format 1.0.

A software card describes an application or a game. It has its releases, each with
a download for every platform. Its page lists the versions, newest first,
with a build per platform and architecture.

Use `"type": "application"` for an application and `"type": "game"` for a game.
Both read the same; libraries can give each a tab of their own.

## An example

```json
{
  "card_version": "1.0",
  "kind": "package",
  "type": "game",
  "id": "io.indiestudio.shadow-quest",
  "name": "Shadow Quest",
  "vendor": {"name": "Indie Studio A"},
  "summary": "Turn-based fantasy RPG with an open world.",
  "description": {"format": "text", "path": "about.txt"},
  "classification": {
    "categories": ["Role Playing", "Adventure"],
    "genres": ["RPG", "Turn-Based"],
    "tags": ["single-player", "moddable"],
    "license": "Proprietary"
  },
  "media": {
    "thumbnail": "media/thumb.png",
    "screenshots": [{"src": "media/shot-1.png", "caption": "The Ashen Vale"}]
  },
  "versions": [
    {
      "tag": "1.4.2",
      "channel": "stable",
      "released": "2026-09-30",
      "summary": "Bug fixes and faster loading.",
      "builds": [
        {"platform": "windows", "arch": "x64",   "min_os": "10",   "file": "files/shadow-quest-1.4.2-windows-x64.zip", "sha256": "..."},
        {"platform": "macos",   "arch": "arm64", "min_os": "12.0", "file": "files/shadow-quest-1.4.2-macos-arm64.zip", "sha256": "..."}
      ]
    },
    {
      "tag": "1.3.5",
      "channel": "legacy",
      "released": "2026-03-08",
      "deprecated": true,
      "deprecation_notice": "Lacks the security fixes in 1.4.0.",
      "builds": [
        {"platform": "windows", "arch": "x64", "file": "files/shadow-quest-1.3.5-windows-x64.zip", "sha256": "..."}
      ]
    }
  ],
  "requirements": {"cpu": "Quad core 2.5 GHz", "ram_mb": 8192, "gpu": "Vulkan 1.2", "disk_mb": 4200},
  "links": [{"label": "Homepage", "url": "https://example.invalid/shadowquest"}]
}
```

## Versions

List them **newest first**, as they should appear. Zephyr never re-sorts them
by number.

| Field | What it is |
|---|---|
| `tag` | The version, as written: `1.4.2`, `2.2.0-beta1` |
| `channel` | `stable`, `beta` or `legacy` (shown as written if anything else) |
| `released` | `YYYY-MM-DD` |
| `summary` | What changed, shown when the version is opened |
| `deprecated` | `true` to warn against it; it stays downloadable |
| `deprecation_notice` | Why, shown in a caution box |
| `builds` | The downloads, below |
| `requirements` | This version's own, when they differ from the card's |

**Which version is current:** the one whose `tag` the card's
`current_version` names, else the first. Use `current_version` to list a beta
above the release that's still current:

```json
"current_version": "2.1.0"
```

The page shows three versions, with the rest behind "Older versions". The
library sorts by the current version's date and shows its size.

## Builds

| Field | What it is |
|---|---|
| `platform` | The system it runs on, from the list below |
| `arch` | `x64`, `x86`, `arm64`, `ppc`, `68k`, `mips`, `sparc`, `6502`, `z80`, `any` |
| `min_os` | The oldest system it runs on, shown as "12.0+": `10`, `6.0.8`, `7.5`, `3.1` |
| `max_os` | Optional: the newest system it runs on, shown as "7.5–9.2". Leave it out when it still runs on the newest |
| `file` | The path to the download |
| `sha256` | Its checksum, see [Checksums](README.md#checksums) |

| `platform` | Shown as | Covers |
|---|---|---|
| `macos` | macOS | Mac OS X and macOS |
| `classic` | Classic Mac OS | System 1 through Mac OS 9; give the oldest in `min_os` (`6.0.8`, `7.5`, `8.6`) |
| `windows` | Windows | Windows 3.x onward; give the oldest in `min_os` (`3.1`, `95`, `10`) |
| `dos` | DOS | MS-DOS and compatibles |
| `linux` | Linux | Give the glibc it needs in `min_os` |
| `amiga` | Amiga | |
| `atari-st` | Atari ST | |
| `apple-ii` | Apple II | |
| `c64` | Commodore 64 | Software for the C64, which also runs on a C128 in C64 mode |
| `c128` | Commodore 128 | Software for the C128's own mode |
| `vic-20` | VIC-20 | |
| `pet` | Commodore PET | |
| `plus4` | Commodore Plus/4 | |
| `atari-8bit` | Atari 8-bit | The 400, 800, XL and XE |
| `zx-spectrum` | ZX Spectrum | |
| `zx81` | ZX81 | |
| `ql` | Sinclair QL | |
| `os2` | OS/2 | |
| `beos` | BeOS | Software for BeOS, which Haiku runs too (x86) |
| `haiku` | Haiku | Software for Haiku only |
| `irix` | IRIX | |
| `solaris` | Solaris | SunOS 4 too |

Other `platform` and `arch` values are shown as written, so spell a system
the same way on every card: the filters count `Amiga` and `amiga` apart.

### Runs on

Libraries can filter by the systems software runs on ("Runs on": System 7,
Mac OS 9, Windows XP). Zephyr works this out from each build's `min_os` and
`max_os`, so write them so they can be read:

- **Mac OS, classic Mac OS, DOS, Amiga, Atari ST, BeOS and IRIX:** the
  version number, alone or after a name: `7.5`, `System 6.0.8`,
  `Mac OS 8.6`, `10.4`, `12.0`, `MS-DOS 5.0`, `Kickstart 1.3`,
  `Workbench 3.1`, `TOS 1.04`, `R5`, `IRIX 6.5`. `Haiku` reads as itself.
- **Linux:** the glibc it needs: `glibc 2.17`. A distribution's version
  (`Ubuntu 22.04`) isn't read, since it doesn't say which glibc.
- **Solaris:** `Solaris 8`, `8` or `2.6`; `SunOS 5.8` reads as Solaris 8
  and `SunOS 4.1.4` as SunOS 4.
- **Windows:** the release: `3.1`, `95`, `98`, `ME`, `2000`, `XP`, `Vista`,
  `7`, `8.1`, `10`, `11`, alone or after "Windows".
- **ZX Spectrum, Atari 8-bit and Commodore 64:** the oldest model it needs:
  `16K`, `48K`, `128K`, `+2`, `+3`; `800`, `800XL`, `130XE`; `C64` or
  `C128`. A Commodore 64 build with `C64` runs on a C128 too.
- **Other systems** (Apple II, VIC-20, ZX81, Sinclair QL, OS/2...) are
  shown as written, but aren't in the Runs on filter.

A build runs on every system from its `min_os` up to its `max_os`, or the
newest when there's none. Old Mac builds stop by themselves where Apple
stopped running them: PowerPC (`ppc`) builds after Mac OS X 10.6, and 32-bit
Intel (`x86`) builds after macOS 10.14, Amiga `68k` builds after
AmigaOS 3.x, and BeOS PowerPC builds after R5 (Haiku runs x86 only). A build with no `min_os`, or one that can't be read, isn't
counted under any system.

**One file for two processors.** A universal build (Intel and Apple silicon,
or 68k and PowerPC) is one file. List it once for each `arch` with the same
`file`: Zephyr counts it once in the size, and Download picks it on either
kind of Mac. **Download *version***
picks the `macos` build for the Mac's own processor, `arm64` or `x64`, or a
build marked `any`. Classic Mac OS builds are offered to download, never as
this Mac's.

## Requirements

```json
"requirements": {"cpu": "Dual core", "ram_mb": 512, "gpu": null, "disk_mb": 260,
                 "notes": "First launch migrates the local database."}
```

`ram_mb` and `disk_mb` are megabytes; Zephyr shows 8192 as "8 GB". For
less than a megabyte, or a size between megabytes, give `ram_kb` or `disk_kb`
in kilobytes instead: 640 KB of memory, or an 800K floppy.

```json
"requirements": {"cpu": "68000", "ram_kb": 1024, "disk_kb": 800, "notes": "Runs from a floppy."}
```

Zephyr shows the largest unit that reads well: "640 KB", "1.5 MB", "8 GB".

Old software often needs particular hardware or other software. Say so in
these, each shown in the Requirements box only when given:

| Field | What it is |
|---|---|
| `display` | The screen it needs: `"VGA 256 colours"`, `"B&W 512×342"` |
| `sound` | Sound hardware it plays through: `["AdLib", "Sound Blaster", "Roland MT-32"]` |
| `input` | What it's played or used with: `["keyboard", "joystick"]` |
| `software` | Other software it needs: `["QuickTime 4", "CarbonLib"]`, `["Rosetta 2"]` |

```json
"requirements": {"cpu": "80386", "ram_kb": 640, "display": "VGA 256 colours",
                 "sound": ["AdLib", "Sound Blaster"], "input": ["keyboard", "joystick"]}
```
Where a card gives both, the `_kb` value is the one shown. A version's own
`requirements` take the place of the card's for that version.

## One version only

An application with a single release can skip `versions` and give its fields at the
top of the card:

```json
"version": "2.3.0",
"channel": "stable",
"released": "2026-09-19",
"builds": [{"platform": "macos", "arch": "arm64", "file": "files/backupsync-2.3.0-macos.zip"}]
```

## Players

A game can say how many play, and how:

```json
"players": 8,
"multiplayer": ["online", "lan"]
```

| Field | What it is |
|---|---|
| `players` | The most players at once. Shown as "Up to 8 players", and filtered in bands: 1, 2, 3–4, 5 or more |
| `multiplayer` | How several play: `hotseat` (taking turns at one computer), `split-screen`, `lan`, `online` or `play-by-email`. Anything else is shown as written |

Leave both out of a game for one player only, or give `"players": 1`.

On a game, `vendor` is shown as the Developer. Name a different publisher in
`publisher`, as any card can.

## Categories, genres and licenses

Filters count values exactly as written, so a library reads best when its
cards agree on them. These are suggestions, not rules.

**Games.** Put the basic genre in `categories`, the finer genre in
`genres`, and the rest in `tags`:

| Field | Suggested values |
|---|---|
| `categories` | Action, Adventure, Role-Playing, Simulation, Strategy, Sports, Racing, Puzzle, Educational |
| `genres` | Platformer, Shooter, Point-and-Click, Roguelike, City Builder, Flight Simulator... |
| `tags` | The perspective (first-person, top-down, side view), the setting (fantasy, sci-fi, historical), the theme (detective, horror, comedy) and the pace (turn-based, real-time) |

**Applications.** `categories`: Utilities, Internet, Graphics, Audio,
Video, Development, Productivity, Education, or System Extensions & Control
Panels for classic Mac add-ons.

**Licenses.** `license`: Commercial, Shareware, Freeware, Demo or Public
Domain, or for open source its [SPDX identifier](https://spdx.org/licenses/):
`MIT`, `GPL-3.0`, `BSD-2-Clause`.

A game or application can also name its `series`, its `languages` and an age
`rating`, as any card can (see [Fields every card has](README.md#fields-every-card-has)).

## Filters for software

Library filters for software are `channel`, `platform` and `arch`, plus
`category`, `genre`, `tag`, `license`, `vendor`, `publisher`, `players`,
`multiplayer`, `series`, `language` and `rating` (see
[Libraries](Libraries.md#filters)).
