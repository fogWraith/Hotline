# Cards: a guide for authors

> Last updated: November 11, 2026. Card format 1.0.

A **card** is a small text file that describes something on a Hotline
server: an application, an album, a video or a series, a book. It sits in the same
folder as the files it describes. Zephyr reads it and shows a proper page:

- a picture and a description;
- what there is to download, with sizes;
- for music, video and books, a button that plays or opens it in Zephyr.

A folder of cards with an `index.card` in it becomes a **library**: one
window with every item, tabs, filters and a search.

Cards need nothing from the server. Any Hotline server can host them, and to
the server they are ordinary files. Clients other than Zephyr list them as
files like any other.

## On a server with a card store

A server with a card store (Janus 2.0.29 and later) can also keep cards in a
database of its own and answer for them, so even a library of thousands
opens at once. Its operator imports a folder of `.card` files, or a
**curator** writes cards on the server directly. A card in a store is the
same card, read by the same rules; its files still live in its folder.

In Zephyr, a curator (an account that may manage cards or libraries) uses
the **Card Editor**: New Card... and Edit Library... in a library's window,
Edit Card... on an item's right-click menu, and New Library... in Hotline →
Show Libraries. The editor shows the card's text beside a preview of its
page, checks it as you type, and says where it doesn't read.

## The guides

| Guide | For |
|---|---|
| This page | What every card has: the file, the shared fields, paths, pictures, checksums |
| [Libraries](Libraries.md) | `index.card`: a library's title, tabs, filters and order |
| [Software](Software.md) | Apps and games, with versions and builds per platform |
| [Albums](Albums.md) | Albums and EPs, with tracks in one or more formats |
| [Videos](Videos.md) | A single video, or a series of episodes |
| [Books](Books.md) | Novels, stories and fan works, whole or a chapter at a time |

Every example in these guides is a working card. For a complete set, every
kind with its index cards, see the documents in the
[conformance set](../Protocol/cards-conformance.json).

## How people open a card

In Zephyr's Files window, select a card and choose **View**:

- **`index.card`** opens the library over its folder.
- **Any other card** opens that item's page.

A card only opens for someone whose account may download, as any View needs.

## A card file

- Any name ending in **`.card`**, such as `night-signals.card`. Upper or lower
  case doesn't matter.
- **Strict JSON**, saved as **UTF-8**: no comments, and no comma after the
  last item of a list.
- **At most 128 KB.** Long text belongs in a separate file (see
  [Descriptions](#descriptions)).

The smallest useful card:

```json
{
  "card_version": "1.0",
  "kind": "package",
  "type": "application",
  "name": "BackupSync",
  "summary": "Automated backup with incremental sync and version history.",
  "version": "2.3.0",
  "released": "2026-09-19",
  "builds": [
    {"platform": "macos", "arch": "arm64", "file": "files/backupsync-2.3.0-macos.zip"}
  ]
}
```

## An item's folder

Keep each item in a folder of its own, with its card at the top:

```
Software/
├── index.card                  ← the library (optional)
└── BackupSync/
    ├── backupsync.card
    ├── about.txt               ← the description
    ├── media/
    │   ├── thumb.png           ← the picture
    │   └── shot-1.png          ← screenshots
    └── files/
        └── backupsync-2.3.0-macos.zip
```

The folder names are up to you. Only the paths in the card have to match.

## Fields every card has

| Field | What it is |
|---|---|
| `card_version` | Always `"1.0"` |
| `kind` | `"package"` for an item (`"index"` is a library's, see [Libraries](Libraries.md)) |
| `type` | `"application"`, `"game"`, `"album"`, `"video"`, `"book"`, or `"other"` |
| `name` | What it's called. Without one, the file name stands in |
| `aka` | Optional: other names it's known by, such as a regional title: `["Schattensuche"]`. Shown under the name, and found by the search |
| `summary` | One sentence, shown under the name and in the library |
| `vendor` | Who made it: `{"name": "The Lantern Rooms"}`. Shown as Vendor, Developer, Artist, Creator or Author by type |
| `publisher` | Optional: who published it, where that isn't who made it: `{"name": "Northwind Interactive"}`. Shown as "Published by" |
| `id` | Optional: your own name for the item, such as `"com.example.backupsync"`. Nothing relies on it being unique |
| `released` | The date it came out, as `YYYY-MM-DD` (software dates its versions instead) |
| `series` | Optional: the series it belongs to, and its place in it: `{"name": "King's Quest", "number": 5}`. Each item in a series is its own card |
| `languages` | Optional: the languages it's in, as codes: `["en", "de", "ja"]`. Shown by name |
| `rating` | Optional: a content or age rating as printed or as you rate it: `PEGI 12`, `ESRB T`, `Teen` |
| `description` | The longer text: `{"format": "text", "path": "about.txt"}` |
| `classification` | `categories`, `genres` and `tags` (lists of words), and `license` |
| `media` | The picture (`thumbnail`) and `screenshots`, see [Pictures](#pictures) |
| `releases` | Optional: its editions, for archivists, see [Releases](#releases) |
| `related` | Optional: other cards this one names, see [Related cards](#related-cards) |
| `links` | Web pages, each a label and an address: `[{"label": "Bandcamp", "url": "https://..."}]`, shown in this order. A label may be used twice |

Each type then has a section of its own, explained in its guide. Fields Zephyr
doesn't know are ignored, so a card can carry more than this.

Two cards may say the same thing: two games called *Doom*, two albums called
*Heroes*, the same `id`. They are still two items, each shown on its own.

## Paths

Every file a card names is given as a path from the card's own folder:

- `media/thumb.png`, `files/backupsync-2.3.0-macos.zip`, `about.txt`.
- Use `/` between folders, never `\`.
- A path must stay inside the card's folder. Zephyr refuses absolute paths
  (`/etc/...`), drive letters (`C:`) and any `..` that leads out of it, and
  shows that file as not offered.
- Spaces and other characters are fine: `flac/01 Carrier Wave.flac`.

Zephyr checks each file against the server's listing. A file the card names
but the server doesn't have is shown struck through and can't be downloaded,
so a missing file is easy to spot.

## Pictures

```json
"media": {
  "thumbnail": "media/thumb.png",
  "screenshots": [
    {"src": "media/shot-1.png", "caption": "Kanban board"},
    {"src": "media/shot-2.png", "caption": "Timeline"}
  ]
}
```

- **`thumbnail`** is the item's picture: an album's cover, a video's poster,
  a book's cover. Its shape goes by the kind of card:

  | Card | Shape | Recommended | Smallest that stays sharp |
  |---|---|---|---|
  | Software (`game`, `application`, `other`) | Square | 512 × 512 | 256 × 256 |
  | Album | Square | 512 × 512 | 256 × 256 |
  | Video | 16:9 | 640 × 360 | 320 × 180 |
  | Book | 2:3 | 512 × 768 | 256 × 384 |

  A picture of another shape is fitted, not cropped, so it shows with bars.
  Larger than the recommended size gains nothing: the picture is shown at
  most 160 points wide (320 pixels on a Retina screen), and it is fetched for
  every item in view, so keep it small, under about 200 KB. JPEG suits
  photographs, PNG drawn art.
- **An episode's `thumbnail`** is 16:9: 320 × 180 is plenty, 160 × 90 the least.
- **`screenshots`** show on the page in the order given, up to 12. A click
  opens one full size. On an album they're headed "Photos". The strip draws
  them 16:9, so that shape fits best; 1280 × 720 or the screen's own size
  reads well full size, and 640 × 360 is the least for a sharp strip.
- **Formats:** PNG, JPEG, GIF, WebP, BMP, TIFF.
- **Size:** at most 4096 pixels a side and 16 MB a file.

## Descriptions

```json
"description": {"format": "text", "path": "about.txt"}
```

The description is a separate file, so the card stays short and the text is
easy to edit. Up to 256 KB. In plain text, a blank line starts a new paragraph.

For bold, italic and the like, write it in Markdown and say so:

```json
"description": {"format": "markdown", "path": "about.md"}
```

(A file ending `.md` is read as Markdown even without the `format`.) Headings,
lists, quotes, `code`, code blocks and rules all show, in the book face.
A description is for drawing attention, not a web page, so pictures aren't
shown (their alt text is), only web addresses are links, and HTML is shown as
typed.

## Checksums

Every file a card offers can carry its **SHA-256**:

```json
{"platform": "windows", "arch": "x64", "file": "files/app-2.1.0-win.zip",
 "sha256": "3ee6265e593903f3bbda551bf8a608e207c8677d91c0f839c67e031977ca4571"}
```

When a download finishes, Zephyr checks it against this. If it doesn't match,
the file is removed and Tasks shows both hashes. A file without a `sha256`
still downloads, marked "not checked". Each item's **Checksums...** window
lists every hash the card gives.

To get a file's SHA-256:

| System | Command |
|---|---|
| macOS | `shasum -a 256 file.zip` |
| Linux | `sha256sum file.zip` |
| Windows (PowerShell) | `Get-FileHash file.zip` |

Give the 64 characters in lowercase. **A checksum must be redone whenever the
file changes:** a stale one makes every download fail.

## Releases

For archivists: the editions an item came out in, physical or regional. Give
each what you know and leave out the rest. The page shows only what's given,
under the heading Releases.

```json
"releases": [
  {"date": "1995-06", "regions": ["United States", "Canada"], "publisher": "Sierra On-Line",
   "media": ["3.5\" floppy", "CD-ROM"], "product_code": "SIE-4711", "notes": "Big box edition with cloth map"},
  {"date": "1996", "regions": ["Germany"], "media": ["CD-ROM"]}
]
```

| Field | What it is |
|---|---|
| `date` | When it came out: `1995`, `1995-06` or `1995-06-04` |
| `regions` | Where: countries or regions, as you'd write them |
| `publisher` | Who published this edition |
| `media` | What it came on: `3.5" floppy`, `5.25" floppy`, `CD-ROM`, `DVD`, `cartridge` |
| `product_code` | A catalog number, UPC or ISBN, as printed |
| `notes` | Anything else about the edition |

Releases describe editions; they don't offer files, and they don't change the
item's date, filters or search. Downloads stay in `versions` or the type's own
files.

## Related cards

A card can point to other cards: the game an expansion is for, or what an
application was called before it was renamed. Transmit's card can say it was
once Transit:

```json
"related": [
  {"relation": "formerly", "card": "../Transit/transit.card", "name": "Transit"}
]
```

| Field | What it is |
|---|---|
| `card` | The path from this card's folder to the other `.card` file. Unlike other paths, it may climb out of this card's folder with `..`, to anywhere on the server |
| `name` | What the link reads. Without one, the file's name |
| `relation` | How they're related, from the list below. Without one, the link reads "See also" |

| `relation` | Shown as | For |
|---|---|---|
| `formerly` | Formerly | The same item under its earlier name |
| `renamed-to` | Renamed to | The same item under its later name |
| `expansion-of` | Expansion for | An expansion or add-on, pointing to what it's for |
| `requires` | Requires | Something it needs |
| `successor-of` | Follows | A sequel, or the version before a big change |
| `remaster-of` | Remaster of | A remaster or remake |
| `port-of` | Port of | A port to another system |
| `part-of` | Part of | A compilation it's in |

The links appear under Related beside the card. Clicking one opens that card
in a window of its own, whatever library it's in. A link shows only on the
card that gives it: if Transit's card should mention Transmit, give it a
`renamed-to` link too. Expansions and add-ons stay items of their own in the
library.

A link to a file that isn't a `.card`, to an `index.card`, or out of the
server's file area is shown by its name and opens nothing.

## Dates, sizes and lengths

- **Dates** are `YYYY-MM-DD`: `"2026-09-30"`. Zephyr shows them as "Sep 30, 2026".
- **Sizes** come from the server, never the card, so they're always right.
- **Lengths** of tracks and videos are whole seconds: `245` shows as 4:05.

## Limits

| | At most |
|---|---|
| A card | 128 KB, nested 16 levels deep, texts of 8 KB |
| Screenshots | 12 |
| Releases | 50 |
| Related cards | 50 |
| Software | 100 versions, 32 builds each |
| Albums | 200 tracks |
| Videos | 200 episodes |
| Books | 1,000 chapters |
| Files per track, episode, chapter or work | 8 |
| A library | folders 6 deep, 500 folders, 2,000 files |

A card past a limit opens on a page naming the limit, rather than half-shown.

## When something is wrong

Zephyr shows a card's problem instead of hiding it:

| Problem | What you see |
|---|---|
| Not valid JSON | "Not a valid card", with the line and column, and the start of the file |
| Neither `kind` nor `type` | "This file ... isn't a card" |
| Too large or too complex | The limit it's past |
| A file not on the server | That file struck through, "missing on the server" |
| A path outside the card's folder | That file "not offered" |
| A checksum that doesn't match | The download fails and is removed; Tasks shows both hashes |
| A newer `card_version` | A note, and what Zephyr can read of it |

The most common mistake is a comma after a list's last item. The second is a
path whose spelling doesn't match the file on the server.
