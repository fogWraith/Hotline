# Card Format 1.0 – Reading Rules

> Last updated: November 11, 2026

> **Conformance language:** The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

A **card** is a small JSON document that describes an item on a Hotline server: software, an album, a video or a series, or a book. A **library** shows many cards as one window, with tabs, filters, a search and a sort.

This document says how a card is read and what a library derives from it: names, dates, filter values, sizes, the search and the orders. It is written for implementers. Authors write cards from the [card authors' guide](../../Cards/README.md), which this document agrees with.

The rules are shared by both ways of showing a library:

- **The walk.** A client reads the `.card` files in a folder and the folders inside it, on any server. Nothing is asked of the server but file listings and downloads.
- **The card store.** A server that implements the [Card Store Extension](Capabilities-Cards.md) keeps cards in a database and answers queries itself.

**A library MUST read the same way whichever shows it.** The same cards and files give the same items, in the same order, with the same filter counts and search results. [Conformance](#conformance) gives the cases that check this.

## Table of Contents

- [The Document](#the-document)
- [Reading Values](#reading-values)
- [Package Cards](#package-cards)
- [Each Type's Section](#each-types-section)
- [Index Cards](#index-cards)
- [Paths](#paths)
- [Related Cards](#related-cards)
- [What a Library Derives](#what-a-library-derives)
- [Picking Items](#picking-items)
- [The Walk](#the-walk)
- [Conformance](#conformance)

---

## The Document

A card is a JSON text ([RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259)) in UTF-8, with no byte order mark. A document that is not valid JSON is **malformed**; a reader reports where (line and column) and why.

A reader MUST refuse a document as **too complex** when:

| Limit | Value |
|---|---|
| Size of the document | more than 131,072 bytes (128 KB) |
| Nesting | a value deeper than 16 levels. The top-level value is at level 1; each array element or object member value is one level deeper than its container |
| A string value | longer than 8,192 bytes of UTF-8. Object keys are not counted |
| `versions` | more than 100 elements |
| `builds`, in any one version | more than 32 elements |
| `media.screenshots` | more than 12 elements |
| `releases` | more than 50 elements |
| `related` | more than 50 elements |
| `tracks` | more than 200 elements |
| `episodes` | more than 200 elements |
| `chapters` | more than 1,000 elements |
| Any one list of files (`files`, `downloads`) | more than 8 elements |

A limit counts every element of the array, of whatever type. These limits are checked only where they apply: `tracks` is counted only on an album, for instance. Which limit a reader reports first, when several are broken, is not specified.

The document's top-level value decides what it is:

1. Not an object: **not a card.**
2. `kind` is the string `"index"`: an [index card](#index-cards).
3. `kind` is the string `"package"`: a [package card](#package-cards).
4. `kind` is absent or not a string, and `type` is a string: a package card.
5. Anything else: **not a card.**

A **duplicate key** in an object is not an error; the last value wins.

**Unknown keys are ignored.** A future format adds keys, and a reader of 1.0 reads what it knows.

**Nothing a card says is unique.** Different items share names, vendors, ids and link labels: there is more than one game called *Doom* and more than one album called *Heroes*. A reader never tells cards apart, merges them or replaces one with another by anything a card says. In a walk a card is its file; in a card store, its key.

**`card_version`** is read as a [text](#reading-values). When it is present and the number before its first `.` is a whole number greater than 1, the card is **newer** than this format. A reader still reads it by these rules, and a client SHOULD say that the card may show incompletely.

## Reading Values

Each value a card gives is read in one of these ways. **A value of the wrong JSON type is absent, never an error.** A card whose `name` is a number has no name, and its other values are still read.

| Reading | Rule |
|---|---|
| **Text** | A string, with leading and trailing white space (the Unicode `White_Space` property) removed. An empty result is absent |
| **Texts** | An array. Each element that is a string is read as a text; other elements and empty results are dropped. Absent or not an array: empty |
| **Whole number** | A JSON number written with digits only: no sign, fraction or exponent, and no greater than 2<sup>64</sup>−1. Anything else is absent |
| **Small number** | A whole number no greater than 2<sup>32</sup>−1 |
| **Flag** | `true` or `false` |
| **Path** | A string, **taken exactly as written** (not trimmed), then resolved as in [Paths](#paths) |
| **Checksum** | A text of exactly 64 characters, all hexadecimal digits, read in lower case. Anything else is absent |
| **Lowered text** | A text with the ASCII letters A–Z made lower case; other characters are left alone |

An object member read from a value that is not an object (`vendor.name` when `vendor` is a string) is absent.

## Package Cards

| Key | Read as | Meaning |
|---|---|---|
| `id` | Text | The author's own identifier. Another item may have the same one |
| `type` | Text | The **kind**: `game`, `application`, `album`, `video` or `book`. Any other text, or none, is the kind `other` |
| `name` | Text | The item's name |
| `aka` | Texts | Other names it is known by, such as a regional title |
| `summary` | Text | One line about it. Absent reads as the empty text |
| `vendor.name` | Text | Who made it |
| `publisher.name` | Text | Who published it, where that isn't who made it |
| `description.path` | Path | A file describing it at length |
| `description.format` | Text | `markdown` (any case) for a Markdown file; anything else is plain text. Absent: Markdown when the path's last name ends `.md` (any case), else plain text. See [Descriptions](#descriptions) |
| `classification.categories` | Texts | |
| `classification.genres` | Texts | |
| `classification.tags` | Texts | |
| `classification.license` | Text | |
| `media.thumbnail` | Path | Its picture |
| `media.screenshots` | Array | Each element that is an object whose `src` is a string becomes a screenshot: `src` (Path) and `caption` (Text). Other elements are dropped |
| `released` | Text | The card's date, `YYYY-MM-DD` |
| `series` | Object | `name` (Text) and `number` (Small number): the series the item belongs to, and its place in it. No name: no series |
| `languages` | Texts | Language codes: `en`, `de`, `ja` |
| `rating` | Text | A content or age rating as the author writes it: `Teen`, `PEGI 12`. Never checked against a scheme |
| `requirements` | Object | `cpu` (Text), `ram_mb` (Whole number), `ram_kb` (Whole number), `gpu` (Text), `display` (Text), `sound` (Texts), `disk_mb` (Whole number), `disk_kb` (Whole number), `input` (Texts), `software` (Texts), `notes` (Text). Where both `ram_kb` and `ram_mb` are given, `ram_kb` is the memory; the same for disk. Requirements are shown, never filtered or searched. Not an object: none |
| `releases` | Array | Editions, **in the array's order**. Each element that is an object becomes a release: `date` (Text: `YYYY`, `YYYY-MM` or `YYYY-MM-DD`), `regions` (Texts), `publisher` (Text), `media` (Texts), `product_code` (Text) and `notes` (Text). A release with none of these is dropped, and so are other elements. Releases are shown, never filtered, searched or dated by. Not an array: none |
| `related` | Array | Other cards this one names, **in the array's order**. Each element that is an object whose `card` is a string becomes a link: `card` (a [related path](#related-cards), taken exactly as written), `name` (Text) and `relation` (Text). Other elements are dropped. Not an array: none |
| `links` | Array | Web pages, **in the array's order**. Each element that is an object with a `label` (Text) and a `url` (Text) becomes a link; other elements are dropped. Labels may repeat: two reviews are two links. Not an array: none |

### Descriptions

A description is text, never a page. A plain one is shown as written, a blank line starting a paragraph. A Markdown one is shown with its headings, paragraphs, lists, quotes, code, rules, and bold, italic and code in its text, and these limits:

- **Pictures are not shown.** An image's alt text stands in its place.
- **Only web links are links**: an `http` or `https` address, followed as the card's links are. Any other link (a file beside the card, a folder, another scheme) is its text alone.
- **HTML is not read.** It is shown as typed.
- What a reader doesn't lay out (tables, footnotes, task lists) is shown as text.

A server never reads a description: it only reports whether the file exists, and its size.

Then the card's kind decides which section is read. [Each Type's Section](#each-types-section) gives them. Only that section is read: an album's `versions` are ignored, for instance.

## Each Type's Section

### Files in formats

Albums, videos and books offer files in formats. A **file list** is an array. Each element that is an object whose `file` is a string becomes a file:

| Key | Read as |
|---|---|
| `file` | Path |
| `format` | Lowered text |
| `quality` | Text |
| `label` | Text |
| `sha256` | Checksum |

Other elements are dropped. A file list absent, or not an array, is empty.

### Software: `game`, `application`, `other`

The versions are read in this order of preference:

1. **`versions` is an array.** Each element that is an object is a version, with its tag read from `tag`. Other elements are dropped.
2. **Otherwise, the short form.** If the card has a `version` member (of any type), or `builds` is an array, the card itself is one version, with its tag read from `version`.
3. **Otherwise,** no versions.

A version:

| Key | Read as | Notes |
|---|---|---|
| `tag` (or `version`, in the short form) | Text | Absent reads as `—` (U+2014) |
| `channel` | Text | `stable`, `beta`, `legacy`, or the author's own |
| `released` | Text | |
| `summary` | Text | Not read in the short form |
| `deprecated` | Flag | Absent is `false` |
| `deprecation_notice` | Text | |
| `builds` | Array | Each element that is an object whose `file` is a string becomes a build: `platform` (Text), `arch` (Text), `min_os` (Text), `max_os` (Text), `file` (Path), `sha256` (Checksum) |
| `requirements` | Object | As the card's. Not read in the short form |

The **current version** is the first version whose tag equals the card's `current_version` (Text), or else the first version. A card with no versions has none.

Software also gives, at the top of the card:

| Key | Read as | Notes |
|---|---|---|
| `players` | Small number | The most players at once. `0` is absent |
| `multiplayer` | Texts | How several play: `hotseat`, `split-screen`, `lan`, `online`, `play-by-email`, or the author's own |

### Albums: `album`

| Key | Read as |
|---|---|
| `tracks` | Array. Each element that is an object is a track: `number` (Small number), `disc` (Small number), `title` (Text; absent reads as `Untitled`), `duration` (Whole number, seconds), `files` (File list) |
| `downloads` | File list: the whole album, as archives |
| `form` | Lowered text: `album`, `ep`, `single`, `compilation`, `live`, or the author's own |

### Videos and series: `video`

When `episodes` is an array, the card is a **series**. Each element that is an object is an episode:

| Key | Read as |
|---|---|
| `number`, `season` | Small number |
| `title` | Text; absent reads as `Untitled` |
| `released` | Text |
| `duration` | Whole number, seconds |
| `summary` | Text |
| `thumbnail` | Path |
| `files` | File list |

Otherwise, it is a single **video**: `duration` (Whole number) and `files` (File list).

### Books: `book`

| Key | Read as | Notes |
|---|---|---|
| `form` | Lowered text | |
| `words` | Whole number | Absent: the sum of the chapters' `words`, when any chapter has them |
| `status` | Lowered text | `complete` or `in-progress`, or the author's own |
| `warnings`, `fandoms`, `characters` | Texts | |
| `files` | File list | The whole work |
| `chapters` | Array | Each element that is an object is a chapter: `number` (Small number), `title` (Text; absent reads as `Untitled`), `released` (Text), `words` (Whole number), `files` (File list) |

## Index Cards

An index card lays out a library.

| Key | Read as | Absent or the wrong type |
|---|---|---|
| `title` | Text | The client's choice, such as the folder's name |
| `subtitle` | Text | None |
| `tabs` | Array. Each element that is an object with a `label` (Text) is a tab; its `type` (Text) is a kind, read as `type` is for a package (so an unknown type is `other`). A tab with no `type` shows every kind | The [default tabs](#default-tabs) |
| `facets` | Array. Each string that is a [facet key](#facets) is kept, in order; other elements are dropped | The [default facets](#facets) |
| `sort` | Text: `newest`, `name` or `size` | `newest`, as for any other text |
| `layout` | Text: `list` | `grid`, as for any other text |

An index card whose `tabs` has no element that becomes a tab has the default tabs. An index card whose `facets` is an empty array (or has no known key) offers no facets.

## Paths

Every file a card names is given relative to the card's **home folder**: the folder its `.card` file is in, or, in a card store, the folder the card is stored with.

A path is a string of names separated by `/`. A reader resolves it as follows:

1. **Refused:** an empty path; one starting with `/`; one containing `\`; one whose second character is `:` (it names a drive).
2. Each name is taken in turn:
   - an empty name refuses the path (`a//b`);
   - `.` is skipped;
   - `..` removes the last name taken, and refuses the path if there is none (the path would leave the home folder);
   - any other name is taken.
3. **Refused:** a path that resolves to no names (it names the home folder itself).

A resolved path is a list of names below the home folder. A **refused** path names nothing: it is never fetched, offered or counted, and a client SHOULD say why.

**Whether a file exists, and its size, come from the server's file area, never from the card.** The size is the one Get File Name List (200) reports for that file. A path that resolves to a folder, or to nothing, is **missing**.

## Related Cards

A card may name other cards in `related`: the game an expansion is for, what an application was called before it was renamed, a compilation it is part of. Each is named by the path to its `.card` file.

**A related path** is resolved like a [path](#paths), with two differences:

1. It starts from the card's folder **in the whole file area**: in a walk, the folder the card was found in, from the top of the file area (the server's root, as the session sees it); in a card store, the card's home folder. So `..` may leave the card's home folder. It is refused only where it would leave the top of the file area.
2. It must name a file whose name ends in `.card` (in any case), and not an `index.card`. Anything else is refused.

**Showing.** A link reads as its `relation` and its `name`; without a name, the file's name with a final `.card` removed. A link whose path is refused is shown by name and opens nothing. A link is shown only on the card that states it: a reader never shows the reverse ("renamed to") on the other card. Related cards are never filtered, searched or counted, and a library never hides or groups an item by them.

**Relations** are read as written. These are suggested, and a reader MAY show them in words of its own:

| `relation` | Meaning |
|---|---|
| `formerly` | This item's earlier name: the other card is the same item, before a rename |
| `renamed-to` | The item's later name |
| `expansion-of` | This is an expansion or add-on for the other |
| `requires` | This needs the other to work |
| `successor-of` | This follows the other, as a sequel or a new major version under the same name |
| `remaster-of` | A remaster or remake of the other |
| `port-of` | A port of the other to another system |
| `part-of` | This is part of the other, such as a compilation |

Without a `relation`, a link is simply related ("See also").

**Opening.** A client opens the other card on its own, in a window of its own, whatever library it is in. On a server with a [card store](Capabilities-Cards.md), it asks [Get Card](Capabilities-Cards.md#get-card-1002) for the card at that place first. Where the store keeps no card there, or the server has none, it reads the `.card` file as it reads one a user Views. A card that is missing, or doesn't read as a package card, opens as such a file would.

## What a Library Derives

### The display name

The card's `name`; without one, the card's file name, with a final `.card` (in any case) removed. In a card store, the file name is the one the card was stored with.

### The date

| Kind | Date |
|---|---|
| Software | The current version's `released` |
| Series | The latest of the episodes' `released`, or else the card's `released` |
| Book | The latest of the chapters' `released`, or else the card's `released` |
| Album, video | The card's `released` |

*Latest* compares dates as strings, by code point. A work in chapters is therefore dated by its newest chapter.

### The item size

This is what Download fetches. It is added up from the files that exist:

| Kind | Files |
|---|---|
| Software | Each build of the current version |
| Album | The first of `downloads`; with no downloads, each track's first file |
| Video | Its first file |
| Series | Each episode's first file |
| Book | The first of `files`; with no files, each chapter's first file |

**Each file counts once.** A file named more than once, such as a universal build listed under two architectures, adds its size once, however many builds, tracks, episodes or chapters name it. Two paths are the same file when they resolve to the same names.

Refused and missing files add nothing. When none of them exists, the size is **unknown**.

### The running time

A video's `duration`. A series', the sum of its episodes' `duration`s, and an album's, the sum of its tracks', where any of them has one. Software and books have none.

### Runs on

Which systems a software item runs on, by **family**: a system's major version, or for an 8-bit machine, its model. Each platform with families has an ordered table; the others have none.

| Platform | Families, oldest first, each with the version it starts at |
|---|---|
| `classic` | `System 1–5` (0), `System 6` (6), `System 7` (7), `Mac OS 8` (8), `Mac OS 9` (9) |
| `macos` | `Mac OS X 10.0` (10.0) to `Mac OS X 10.7` (10.7), one a minor version; `OS X 10.8` (10.8) to `OS X 10.11` (10.11); `macOS 10.12` (10.12) to `macOS 10.15` (10.15); `macOS 11` (11), `macOS 12` (12), `macOS 13` (13), `macOS 14` (14), `macOS 15` (15), `macOS 26` (26) |
| `windows` | `Windows 3.x`, `Windows 95`, `Windows 98`, `Windows ME`, `Windows 2000`, `Windows XP`, `Windows Vista`, `Windows 7`, `Windows 8`, `Windows 10`, `Windows 11` |
| `dos` | `DOS 1` (1) to `DOS 6` (6), one a major version |
| `linux` | By its C library: `Linux glibc 2.0` (2.0), `Linux glibc 2.3` (2.3), `Linux glibc 2.5` (2.5), `Linux glibc 2.12` (2.12), `Linux glibc 2.17` (2.17), `Linux glibc 2.27` (2.27), `Linux glibc 2.31` (2.31), `Linux glibc 2.35` (2.35), `Linux glibc 2.39` (2.39) |
| `beos` | `BeOS R3` (3), `BeOS R4` (4), `BeOS R5` (5), then `Haiku`, named |
| `irix` | `IRIX 3` (3), `IRIX 4` (4), `IRIX 5` (5), `IRIX 6` (6), `IRIX 6.5` (6.5) |
| `solaris` | `SunOS 4` (1), `Solaris 2.x` (2), `Solaris 7` (7), `Solaris 8` (8), `Solaris 9` (9), `Solaris 10` (10), `Solaris 11` (11) |
| `amiga` | `AmigaOS 1.x` (1), `AmigaOS 2.x` (2), `AmigaOS 3.x` (3), `AmigaOS 4.x` (4) |
| `atari-st` | `TOS 1.x` (1), `TOS 2.x` (2), `TOS 3.x` (3), `TOS 4.x` (4) |
| `atari-8bit` | `Atari 400/800`, `Atari XL`, `Atari XE` |
| `c64` | `Commodore 64`, `Commodore 128` (in C64 mode) |
| `zx-spectrum` | `ZX Spectrum 16K`, `ZX Spectrum 48K`, `ZX Spectrum 128K`, `ZX Spectrum +2`, `ZX Spectrum +3` |

The 8-bit machines run by model, not by system, and Linux by its C library, which decides whether a program runs where a kernel or distribution version doesn't. Other platforms (`apple-ii`, `c128`, `vic-20`, `pet`, `plus4`, `zx81`, `ql`, `haiku`...) have no families.

**Reading a version.** A build's `min_os` and `max_os` are read as a family of its platform:

1. **A name.** The text is broken into words at anything other than a letter, a digit, `.` or `+`, and the first word that names a family, in any case, gives it:
   - `windows`: `3`, `3.0`, `3.1` and `3.11` are `Windows 3.x`; `95`, `98`, `ME`, `2000`, `XP`, `Vista`, `7`, `8`, `8.1`, `10` and `11` are the releases they name (`8.1` is `Windows 8`).
   - `atari-8bit`: `400` and `800` are `Atari 400/800`; `XL`, `600XL`, `800XL` and `1200XL` are `Atari XL`; `XE`, `65XE`, `130XE`, `800XE` and `XEGS` are `Atari XE`.
   - `c64`: `C64` and `64` are `Commodore 64`; `C128` and `128` are `Commodore 128`.
   - `zx-spectrum`: `16K` and `16`, `48K` and `48`, `128K` and `128` are the models they name; `+2`, `+2A` and `+2B` are `ZX Spectrum +2`; `+3` is `ZX Spectrum +3`.
   - `beos`: `Haiku` is `Haiku`.
2. **Otherwise, a version.** The first run of ASCII digits and `.` in the text, read as numbers separated by dots (`System 7.5` is 7.5, `12.0` is 12, `Mac OS 8.6` is 8.6), and compared number by number. Its family is the last numbered one that starts at or below it. Two platforms read it from part of the text:
   - `linux`: only the text after the first `glibc` (any case); a text without it doesn't read, so a distribution's version (`Ubuntu 22.04`) never stands for glibc's.
   - `solaris`: after a `SunOS` (any case), version 4 is Solaris 1, and 5.*n* is Solaris 2.*n* below 7 and Solaris *n* from 7, so `SunOS 5.6` is Solaris 2.6 and `SunOS 5.8` is Solaris 8.
3. Anything else does not read.

**What a build covers.** Every family of its platform from its `min_os` to the **end**: the earliest of its `max_os`, the cap its processor sets, and the table's last family. The caps: a `macos` build for `ppc` ends at `Mac OS X 10.6` and one for `x86` at `macOS 10.14`, and one for `arm64` starts no earlier than `macOS 11`; an `amiga` build for `68k` ends at `AmigaOS 3.x`; a `beos` build for `ppc` ends at `BeOS R5`, since Haiku runs BeOS's x86 software only. A build whose `min_os` is absent or doesn't read, whose platform has no table, or whose end comes before its start, covers nothing. A `max_os` that doesn't read sets no end.

### Facets

A **facet** groups items by a value. Each key reads an item's values as follows. Values are taken exactly as read (case matters: `RPG` and `rpg` are two values), and each value counts once per item.

| Key | Values |
|---|---|
| `channel` | The current version's `channel` |
| `platform` | Each `platform` of the current version's builds |
| `arch` | Each `arch` of the current version's builds |
| `category` | `classification.categories` |
| `genre` | `classification.genres` |
| `tag` | `classification.tags` |
| `license` | `classification.license` |
| `vendor` | `vendor.name` |
| `publisher` | `publisher.name` |
| `players` | Software's `players`, as a band: `1`, `2`, `3–4` (3 or 4) or `5 and over`. The dash is U+2013 |
| `multiplayer` | Software's `multiplayer` |
| `format` | Each `format` of the card's [media files](#media-files) |
| `quality` | Each `quality` of the card's media files |
| `year` | The first four characters of the [date](#the-date), when they are all ASCII digits |
| `fandom` | A book's `fandoms` |
| `rating` | `rating` |
| `status` | A book's `status` |
| `series` | `series.name` |
| `language` | Each of `languages` |
| `character` | A book's `characters` |
| `words` | A book's `words`, as a band: `Under 10k` (below 10,000), `10k–40k` (below 40,000), `40k–100k` (below 100,000) or `100k and over`. The dash is U+2013 |
| `form` | A book's or an album's `form`; a video's is `video`, and a series' `series` |
| `warning` | A book's `warnings` |
| `duration` | The [running time](#the-running-time), as a band: `Under 10 min` (below 600 seconds), `10–30 min` (below 1,800), `30–60 min` (below 3,600), `1–2 hours` (below 7,200) or `Over 2 hours`. The dashes are U+2013 |
| `decade` | The first three characters of the [date](#the-date) followed by `0s` (`1990s`), when its first four are all ASCII digits |
| `os` | Each family a build of the current version [runs on](#runs-on) |

Facet values come from the card alone. A file that is missing still gives its format and quality.

The **default facets**, for an index card that names none, are `channel`, `platform`, `arch`, `os`, `category`, `genre`, `form`, `players`, `multiplayer`, `vendor`, `publisher`, `series`, `format`, `quality`, `duration`, `decade`, `year`, `fandom`, `rating`, `warning`, `status`, `words` and `language`, in that order.

#### Media files

| Kind | Media files |
|---|---|
| Software | None |
| Album | Every track's files, then `downloads` |
| Video | Its files |
| Series | Every episode's files |
| Book | `files`, then every chapter's files |

### Words and lowering

The search and the name order break text into **words** and **lower** it.

- **A word** is a run of characters that each have the Unicode `Alphabetic` property, or a general category of `Nd`, `Nl` or `No`. Every other character separates words.
- **Lowering** maps each character by its *simple* lowercase mapping in `UnicodeData.txt`, one character at a time. The special and contextual mappings of `SpecialCasing.txt` do not apply, so `İ` lowers to `i` and `Σ` always to `σ`.

Implementations may use different Unicode versions. The conformance cases use only characters whose properties have not changed.

## Picking Items

A library shows the items that **pass** the current **tab**, **ticks** and **search**, in the current **sort**.

### Passing

An item passes when all three of these hold:

1. **The tab.** The tab has no kind, or its kind is the item's.
2. **The ticks.** For every facet key with at least one value ticked, the item has at least one of the ticked values for that key. Ticks *or* within a key and *and* across keys. Ticks may be held for any facet key, offered by the library or not.
3. **The search.** Every word of the search, lowered, begins some word of the item's **search text**, lowered. A search with no words passes every item.

The search text is:
- the display name, the other names (`aka`), the summary, the vendor, the publisher, the tags, the categories, the genres and the series name;
- every track, episode and chapter title;
- a book's fandoms, characters and warnings.

### Order

| Sort | Order |
|---|---|
| `newest` | Date, latest first; items without a date last. Ties by name |
| `name` | Name |
| `size` | Item size, largest first; unknown last. Ties by name |

*By name* compares the lowered display names by code point. Items that are still equal are ordered by home folder (name by name, by code point, a shorter path first where one begins the other), then by file name, by code point.

### Counts

For each facet the library offers, every value present among the items that pass is counted. Items are tested against the tab, the search, and the ticks of **every other** facet. A facet's own ticks don't narrow its own counts.

- A facet with no values is left out.
- Values are listed by code point, with these exceptions:
  - `channel` lists `stable`, `beta` and `legacy` first, in that order, then the rest by code point;
  - `words` and `duration` list their bands shortest first;
  - `os` lists its families in the order of the [tables](#runs-on): `classic`'s, then `macos`', `windows`', `dos`', `linux`', `beos`', `irix`', `solaris`', `amiga`'s, `atari-st`'s, `atari-8bit`'s, `c64`'s and `zx-spectrum`'s.

Each **tab** has a count too: the number of items that pass under that tab, with the same ticks and search.

### Default tabs

A library without tabs of its own has a tab with no kind labelled `All`. It also has a tab for each kind present among its items, but only when there is more than one such kind:

| Kind | Label |
|---|---|
| game | Games |
| application | Applications |
| album | Albums |
| video | Videos |
| book | Books |
| other | Other |

The kind tabs come in the order of this table.

## The Walk

A client walks a library when the server has no card store, or the user Views an `index.card` file directly. The library is the `index.card`'s folder and every folder inside it, and every file whose name ends in `.card` (in any case) other than an `index.card` is a card. An `index.card` further down lays out a library of its own; the outer library still shows that folder's items.

- **Limits.** A client MUST bound its walk, and SHOULD say when it stopped short. Zephyr stops at 6 folders deep, 500 folders or 2,000 files, and fetches cards four at a time.
- **Cards it can't read.** The walk shows a card that is malformed, not a card or too complex as an item, named by its file. Such an item passes only under a tab with no kind, with nothing ticked, and with a search that its file name (as for the display name) passes. It has no facet values, no date and no size. A card store never holds such a card: it refuses it when it is written.
- **Sizes and existence** come from the walk's own listings.

## Conformance

These cases run against a set of fixture libraries, which [`cards-conformance.json`](cards-conformance.json) holds whole: every fixture file with its path and size (so a card store can be tested without the media), each card's document, and the cases below. Each top-level fixture folder is a library with the default tabs and facets unless its `index.card` says otherwise; the root, with every folder in it, is the library the cases call "all fixtures".

### Reading

| Case | Result |
|---|---|
| `Problems/broken.card` | Malformed, at line 3 |
| `Problems/untyped.card` | Not a card |
| `Problems/future.card` | A package, newer than this format, named `From The Future` |
| `Problems/no-downloads.card` | A package with no versions and no current version |
| `Problems/escape.card` | Its thumbnail and every build's file are refused |
| `Software/TaskMaster Pro/taskmaster-pro.card` | Current version `2.1.0` (named by `current_version`, though a beta is listed later) |
| `Software/BackupSync/backupsync.card` | The short form: one version, `2.3.0`, with two builds |
| `Software/Shadow Quest/shadow-quest.card` | Three versions, the third deprecated with a notice; links `Homepage`, `Changelog`, then two labelled `Review` |
| `{"kind":"package","name":5,"links":[1],"classification":{"tags":"x"},"versions":[{"tag":"1","builds":[{"file":7},{"file":"a.zip","sha256":"nothex"}]}]}` | No name, no links, no tags; one build, with no checksum |
| `Books/Second Light` | `words` 9,335 (3,120 + 2,875 + 3,340, from its chapters); status `in-progress`; languages `en`; rating `Mature` |
| `Software/Puzzle Master/puzzle-master.card` | Series `Mindbender Trilogy`, number 3; languages `en`, `de`, `ja`; rating `PEGI 3`; memory 768 KB (`ram_kb` wins over `ram_mb`), disk 1,400 KB; four builds |
| `{"type":"album","series":{"name":"S","number":-1},"languages":"en","rating":7}` | Series `S` with no number; no languages; no rating |
| `Software/Kingdom Builder/kingdom-builder.card` | Publisher `Northwind Interactive`; players 8; multiplayer `online`, `lan` |
| `Software/Shadow Quest/shadow-quest.card` | Other names: `Schattensuche` |
| `{"type":"video","aka":["  ", "Other", 3]}` | Other names: `Other` (a blank name and a number are dropped) |
| `{"type":"album","publisher":{"name":"Label"},"players":4,"multiplayer":["lan"]}` | Publisher `Label`; no players and no multiplayer (only software gives them) |
| `{"type":"game","publisher":"Label","players":0,"multiplayer":"lan"}` | No publisher (not an object), no players (`0` is absent), no multiplayer (not an array) |
| `{"type":"game","series":{"number":2},"requirements":{"ram_mb":4,"disk_mb":20,"disk_kb":"x"}}` | No series (it has no name); memory 4,096 KB; disk 20,480 KB (`disk_kb` is not a whole number, so `disk_mb` stands) |

### Paths

| Declared | Resolved |
|---|---|
| `media/../files/a.zip` | `files`, `a.zip` |
| `./a.png` | `a.png` |
| `""`, `/etc/passwd`, `a//b`, `..`, `a/../../b`, `C:/x`, `files\x.zip`, `.` | Refused |

### Libraries

| Library | Case | Result |
|---|---|---|
| `Software` | Tab counts | All 8, Games 4, Applications 4 |
| `Software` | `newest`, first four | Stellar Drift, TaskMaster Pro, PixelCraft, SecureVault |
| `Software` | `channel` counts | stable 7, beta 1 |
| `Software` | `platform` counts | classic 1, linux 3, macos 8, windows 8 |
| `Software` | `genre` under Applications | Left out (no values) |
| `Software` | Ticked `platform` linux, classic | 4 items |
| `Software` | ...and `channel` beta | Stellar Drift; `platform` windows counts 1 |
| `Software` | Search `pass` | SecureVault |
| `Software` | Search `TURN rpg` | Shadow Quest |
| `Software` | Search `ault` | Nothing (a word's start, not its middle) |
| `Software` | PixelCraft's item size | 436 bytes; its older version's macOS build is missing |
| `Software` | Puzzle Master's item size | Its three files, each once: the universal macOS build is listed for `arm64` and `x64` but counted once |
| `Software` | `series` counts | Mindbender Trilogy 1 |
| `Software` | Search `mindbender` | Puzzle Master (a series name) |
| `Software` | `players` counts | 1 1, 2 1, 3–4 1, 5 and over 1 |
| `Software` | `multiplayer` counts | hotseat 1, lan 1, online 1, split-screen 1 |
| `Software` | `publisher` counts | Northwind Interactive 1 |
| `Software` | Search `northwind` | Kingdom Builder (a publisher) |
| `Software` | Search `schatten` | Shadow Quest (another name) |
| `Software` | `os` counts, for Mac OS 8, Mac OS 9, macOS 11, macOS 13 and Windows 10 | 1, 1, 1, 8 and 8: Puzzle Master's classic build from 8.6, its Mac build from 11.0, every card's Mac build by macOS 13 and Windows build from 10 |
| `Vintage` | `os` counts | System 6 to System 7 (68k, up to 7.6), Mac OS X 10.2 to 10.6 (PowerPC, capped), Windows 95 to XP (up to XP), DOS 5 and DOS 6, Linux glibc 2.3 to 2.12 (up to 2.12), BeOS R5 and Haiku (x86), IRIX 5 to 6.5, Solaris 2.x to 8 (from SunOS 5.6, up to 8), AmigaOS 1.x to 3.x (68k, capped), TOS 1.x to 4.x, Atari XL and XE, Commodore 64 and 128, ZX Spectrum 48K to +3: 1 each, in that order |
| `Vintage` | Ticked `os` Mac OS X 10.4 | Lunar Lander |
| `Vintage` | Ticked `os` Mac OS 8, macOS 10.14, Windows Vista, AmigaOS 4.x, ZX Spectrum 16K, Atari 400/800, Linux glibc 2.17, BeOS R4, Solaris 9 | Nothing: each is outside one of its builds' ranges |
| `Vintage` | `decade` counts | 1980s 1, 1990s 1 |
| (all fixtures) | `duration` counts | Under 10 min 4, 30–60 min 1, in that order |
| `Music` | `form` counts | album 1, ep 1 |
| `Video` | `form` counts | series 1, video 1 |
| `Books` | `form` counts | novel 1, novella 1, short-story 2 |
| `Books` | `warning` counts | Graphic depictions of violence 1, Major character injury 1 |
| `Software` | Ticked `players` 2, 3–4, 5 and over | Kingdom Builder, Puzzle Master, Stellar Drift |
| (all fixtures) | `language` counts | de 1, en 5, ja 1 |
| (all fixtures) | `rating` counts | General 2, Mature 1, PEGI 3 1, Teen 1 |
| (all fixtures) | Default tabs | All, Games, Applications, Albums, Videos, Books |
| `Music` | `newest` | Night Signals, Harbour Lights |
| `Music` | `format` counts | flac 1, mp3 2 |
| `Music` | Harbour Lights' item size | The sum of its tracks' first files that exist (no `downloads`) |
| `Video` | `newest` | Retro Repairs (its newest episode), Building a Hotline Server |
| `Video` | `quality` counts | 1080p 2, 480p 2 (episode 3's refused 1080p still counts) |
| `Video` | Search `powerbook` | Retro Repairs (an episode title) |
| `Books` | `newest` | Second Light, The Lantern Keeper's Daughter, Low Tide, The Drowned Bell |
| `Books` | `words` counts | Under 10k 3, 10k–40k 1, in that order |
| `Books` | `fandom` counts | The Tidewater Cycle - M. Harrow 1 |
| `Books` | Search `tide left` | Second Light first (a chapter title) |
| `Books` | Search `tamsin` | Second Light first (a character) |
| `Books` | Search `injury` | The Lantern Keeper's Daughter first (a warning) |
