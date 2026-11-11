# Video cards

> Part of the [card authors' guide](README.md). Card format 1.0.

A video card describes one video, or a series of episodes.
Play opens a video in Zephyr's movie player, which starts as it arrives.
Download fetches it in the quality chosen.

## One video

```json
{
  "card_version": "1.0",
  "kind": "package",
  "type": "video",
  "id": "example.building-a-hotline-server",
  "name": "Building a Hotline Server",
  "vendor": {"name": "Pixel & Solder"},
  "summary": "From an empty folder to a server people can find.",
  "released": "2026-08-20",
  "duration": 1260,
  "description": {"format": "text", "path": "about.txt"},
  "classification": {"categories": ["Tutorial"], "tags": ["hotline", "servers"]},
  "media": {"thumbnail": "poster.png"},
  "files": [
    {"quality": "1080p", "format": "mp4", "file": "video/building-1080p.mp4", "sha256": "..."},
    {"quality": "480p",  "format": "mp4", "file": "video/building-480p.mp4",  "sha256": "..."}
  ],
  "links": [{"label": "Patreon", "url": "https://example.invalid/pixelandsolder"}]
}
```

The page shows the poster large, with **Play** and **Download**. Download
offers each quality with its size.

## A series

Give `episodes` in place of `duration` and `files`:

```json
{
  "card_version": "1.0",
  "kind": "package",
  "type": "video",
  "name": "Retro Repairs",
  "vendor": {"name": "Pixel & Solder"},
  "summary": "Bringing old Macs back to life, one board at a time.",
  "media": {"thumbnail": "poster.png"},
  "episodes": [
    {
      "number": 1,
      "title": "The SE that wouldn't chime",
      "released": "2026-09-02",
      "duration": 1104,
      "summary": "A Macintosh SE with a dead startup chime.",
      "thumbnail": "thumbs/ep1.png",
      "files": [
        {"quality": "1080p", "format": "mp4", "file": "video/ep1-1080p.mp4", "sha256": "..."},
        {"quality": "480p",  "format": "mp4", "file": "video/ep1-480p.mp4",  "sha256": "..."}
      ]
    }
  ]
}
```

The page lists the episodes, each with its picture, title, length, date,
qualities and summary, and Play and Download buttons. Right-click an episode
for every quality. A series is labelled "Series". The library dates it by
its **newest episode**, so a new episode moves it to the front.

A card with both `files` and `episodes` is read as a series.

| Episode field | What it is |
|---|---|
| `number` | Shown before the title |
| `title` | The episode's name |
| `released` | `YYYY-MM-DD` |
| `duration` | Whole seconds |
| `summary` | One line, under the title |
| `thumbnail` | Its picture, 16:9 |
| `files` | The episode in each quality |
| `season` | Optional. Episodes are listed in the order given |

## Files and qualities

| Field | What it is |
|---|---|
| `quality` | `1080p`, `720p`, `480p`, or as you name it |
| `format` | `mp4`, `mov`, `avi` |
| `file` | The path to it |
| `sha256` | Its checksum |

**Order matters.** Play uses the first listed file Zephyr can play, and an
episode's download button fetches the first the server has. List the quality
to stream first: a smaller one starts sooner on a slow connection.

## Making videos that play well

Zephyr plays the MP4 most tools make: **H.264 video with AAC sound**. Also
QuickTime (`.mov`), AVI, FLV and others.

**Make MP4s "fast start".** An MP4 keeps its index either at the start or at
the end. With it at the end, Zephyr can't start until the whole file has
arrived; with it at the start, the video plays as it downloads. Most upload
presets do this already. With ffmpeg, add `-movflags +faststart`:

```
ffmpeg -i master.mov -c:v libx264 -crf 22 -c:a aac -b:a 160k -movflags +faststart video/ep1-1080p.mp4
```

An existing MP4 can be fixed without re-encoding:

```
ffmpeg -i old.mp4 -c copy -movflags +faststart video/ep1-1080p.mp4
```

## Filters for videos

Library filters for videos are `vendor` (shown as Creator), `category`,
`tag`, `quality`, `format`, `form` (a single video or a series), `duration`
(a series' episodes added up), `year` and `decade` (see
[Libraries](Libraries.md#filters)).
