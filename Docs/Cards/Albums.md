# Album cards

> Part of the [card authors' guide](README.md). Card format 1.0.

An album card describes an album or an EP. Its page lists the tracks the way
a CD player does: number, play button, title, length and download. Play opens
the track in Zephyr's audio player, which starts as the track arrives.
**Download Album** fetches the whole album at once.

## An example

```json
{
  "card_version": "1.0",
  "kind": "package",
  "type": "album",
  "id": "example.night-signals",
  "name": "Night Signals",
  "vendor": {"name": "The Lantern Rooms"},
  "summary": "Synthwave from a lighthouse, recorded over one winter.",
  "released": "2026-09-12",
  "description": {"format": "text", "path": "about.txt"},
  "classification": {"genres": ["Synthwave", "Electronic"], "tags": ["instrumental"], "license": "CC BY-NC 4.0"},
  "media": {
    "thumbnail": "cover.png",
    "screenshots": [{"src": "photos/studio.png", "caption": "The lamp room"}]
  },
  "tracks": [
    {
      "number": 1, "title": "Carrier Wave", "duration": 245,
      "files": [
        {"format": "flac", "file": "flac/01 Carrier Wave.flac", "sha256": "..."},
        {"format": "mp3",  "file": "mp3/01 Carrier Wave.mp3",   "sha256": "..."}
      ]
    },
    {
      "number": 2, "title": "Fog Signal", "duration": 221,
      "files": [
        {"format": "flac", "file": "flac/02 Fog Signal.flac", "sha256": "..."},
        {"format": "mp3",  "file": "mp3/02 Fog Signal.mp3",   "sha256": "..."}
      ]
    }
  ],
  "downloads": [
    {"format": "flac", "label": "Whole album (FLAC)", "file": "night-signals-flac.zip", "sha256": "..."},
    {"format": "mp3",  "label": "Whole album (MP3)",  "file": "night-signals-mp3.zip",  "sha256": "..."}
  ],
  "links": [{"label": "Bandcamp", "url": "https://example.invalid/night-signals"}]
}
```

A folder for it:

```
Night Signals/
├── night-signals.card
├── cover.png
├── about.txt
├── photos/studio.png
├── flac/01 Carrier Wave.flac ...
├── mp3/01 Carrier Wave.mp3 ...
├── night-signals-flac.zip
└── night-signals-mp3.zip
```

## Album, EP or single

`form` says what kind of release it is: `album`, `ep`, `single`,
`compilation` or `live`. Zephyr shows it in place of "Album" ("EP", "Live
album"), and libraries can filter by it. Leave it out for an album.

```json
"form": "ep"
```

## Tracks

| Field | What it is |
|---|---|
| `number` | The track number shown |
| `title` | The track's name |
| `duration` | Its length in whole seconds: `245` shows as 4:05 |
| `files` | The track in each format offered, below |
| `disc` | Optional, for a multi-disc album. Tracks are listed in the order given |

The page lists tracks in the order the card gives them. Its Overview adds up
their lengths.

## A track's files

| Field | What it is |
|---|---|
| `format` | `flac`, `mp3`, `ogg`, `wav`, `aiff` |
| `file` | The path to it |
| `sha256` | Its checksum |
| `label` | Optional: how the format is named, if not just "FLAC" |

**Order matters.** A track's download button fetches its **first listed file
that the server has**: if the FLAC is missing, the MP3 comes instead. Play
uses the first listed file Zephyr can play. Right-clicking a track lists
every format.

Zephyr plays **MP3, FLAC, Ogg Vorbis, WAV and AIFF**. To stream, MP3 starts
fastest; to keep, FLAC. Listing FLAC first and MP3 second suits most
listeners.

## Whole-album downloads

`downloads` lists an archive of the whole album in each format: usually a zip
of the tracks. **Download Album** opens a menu of them, with their sizes.
Without `downloads`, the button reads **Download All Tracks** and fetches
every track in its first format, one at a time.

## Filters for albums

Library filters for albums are `vendor` (shown as Artist), `genre`, `tag`,
`format`, `form`, `duration` (the tracks' lengths added up), `year`,
`decade` and `license` (see [Libraries](Libraries.md#filters)). The library
sorts albums by `released`.
