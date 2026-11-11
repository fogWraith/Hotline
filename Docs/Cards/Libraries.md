# Libraries

> Part of the [card authors' guide](README.md). Card format 1.0.

A library is a folder of cards shown as one window: every item as a
grid or a list, with tabs across the top, filters down the side and a search
field. It opens when someone Views the folder's **`index.card`**.

## What a library includes

Every card in the folder **and in the folders inside it** belongs to the
library, up to 6 folders deep. One library can therefore hold a folder
per item, or a folder per category with items inside.

The search covers names, summaries, makers, genres, categories and tags. It
also covers track, episode and chapter titles, and a book's fandoms,
characters, warnings and series. Each word typed matches the start of a word:
`shop` finds "Shopkeeper".

## The index card

```json
{
  "card_version": "1.0",
  "kind": "index",
  "title": "Software & Games",
  "subtitle": "Games and applications hosted on this server",
  "tabs": [
    {"label": "All"},
    {"label": "Games", "type": "game"},
    {"label": "Applications", "type": "application"}
  ],
  "facets": ["channel", "platform", "arch", "category", "genre"],
  "sort": "newest",
  "layout": "grid"
}
```

| Field | What it does |
|---|---|
| `title` | The window's title; without one, the folder's name |
| `subtitle` | A line under the tabs |
| `tabs` | The tabs, in order. A tab with a `type` shows only that type; one without shows everything. Each tab shows its count |
| `facets` | The filter groups down the side, in order (below) |
| `sort` | How it starts sorted: `"newest"`, `"name"` or `"size"`. The window can change it |
| `layout` | How it starts: `"grid"` or `"list"` |

With a single tab, the tab bar is left out.

## Filters

Each name in `facets` is one group of checkboxes down the side, each with a
count. Ticking several in one group shows items with *any* of them;
ticking in several groups shows items with *all* of them.

**A group no item in the current tab has is left out.** One list can
therefore serve a mixed library: Genre appears under Games, and Fandom under
Books.

| Name | Groups by | From |
|---|---|---|
| `channel` | Stable, Beta, Legacy | Software's current version |
| `platform` | Windows, macOS, Linux, Classic Mac OS, DOS, Amiga... | Software's current builds |
| `arch` | x64, x86, ARM64, PowerPC, 68k | Software's current builds |
| `category` | Categories | `classification.categories` |
| `genre` | Genres | `classification.genres` |
| `tag` | Tags | `classification.tags` |
| `license` | License | `classification.license` |
| `vendor` | Who made it (Vendor, Developer, Artist, Creator or Author, by tab) | `vendor.name` |
| `publisher` | Who published it | `publisher.name` |
| `players` | 1, 2, 3–4, 5 or more players | Software's `players` |
| `multiplayer` | Hotseat, Split screen, LAN, Online, Play by email | Software's `multiplayer` |
| `os` | Runs on: System 7, Mac OS 9, Mac OS X 10.4, Windows XP, DOS 6, Linux glibc 2.17, BeOS R5, IRIX 6.5, Solaris 8, AmigaOS 3.x, TOS 2.x, Commodore 64, ZX Spectrum 48K... | Software's current builds, see [Runs on](Software.md#runs-on) |
| `form` | Novel, EP, Single, Live album, Series, Video... | A book's or an album's `form`; a single video or a series |
| `warning` | Warnings | A book's `warnings` |
| `duration` | Under 10 min, 10–30 min, 30–60 min, 1–2 hours, Over 2 hours | A video's length, or a series' or an album's added up |
| `decade` | 1980s, 1990s... | The item's date, as `year` |
| `format` | File formats (FLAC, MP3, EPUB...) | An album's, video's or book's files |
| `quality` | 1080p, 480p... | A video's files |
| `year` | Year | The item's date |
| `fandom` | Fandoms | A book's `fandoms` |
| `rating` | Ratings | `rating` |
| `status` | Complete, In progress | A book's `status` |
| `words` | Under 10k, 10k–40k, 40k–100k, 100k and over | A book's length |
| `series` | Series | `series` |
| `language` | Languages, by name | `languages` |
| `character` | Characters | A book's `characters` |

Values are matched exactly as written, so keep them consistent across cards:
`RPG` and `rpg` are two different genres.

## Without an index card

A folder of cards with no `index.card` can still be opened by Viewing any of
its cards. That shows one item, not a library. Add an `index.card` to
offer the library. The smallest one:

```json
{"card_version": "1.0", "kind": "index", "title": "Downloads"}
```

Without its own `tabs`, it shows All, plus a tab for each type when the folder
holds more than one. Without `facets`, it offers every group above except
`tag`, `license`, `series` and `character`, which tend to have too many values
to tick through. Name those in `facets` to offer them.

## An index card further down

An `index.card` in a folder inside the library makes that folder a
library of its own, which can be Viewed by itself. The outer library
still shows its items.
