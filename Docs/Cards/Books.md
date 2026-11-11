# Book cards

> Part of the [card authors' guide](README.md). Card format 1.0.

A book card describes a written work: a novel, a novella, a short story or a
fan work. It can offer the whole work, its chapters, or both. **Read** opens
it in Zephyr's own reader, and **Download** saves it.

## An example

```json
{
  "card_version": "1.0",
  "kind": "package",
  "type": "book",
  "id": "example.lantern-keepers-daughter",
  "name": "The Lantern Keeper's Daughter",
  "vendor": {"name": "M. Harrow"},
  "form": "novella",
  "series": {"name": "The Tidewater Cycle", "number": 1},
  "summary": "A keeper's daughter finds a second light on the shoals.",
  "released": "2026-08-14",
  "languages": ["en"],
  "words": 38400,
  "status": "complete",
  "rating": "Teen",
  "warnings": ["Major character injury"],
  "characters": ["Wren Harrow", "Old Tamsin"],
  "description": {"format": "text", "path": "about.txt"},
  "classification": {"genres": ["Fantasy"], "tags": ["found family", "slow burn"], "license": "CC BY-NC-ND 4.0"},
  "media": {"thumbnail": "cover.png"},
  "files": [
    {"format": "epub", "file": "the-lantern-keepers-daughter.epub", "sha256": "..."},
    {"format": "pdf",  "file": "the-lantern-keepers-daughter.pdf",  "sha256": "..."}
  ],
  "links": [{"label": "Ko-fi", "url": "https://example.invalid/mharrow"}]
}
```

Its page reads "by M. Harrow · Book 1 of The Tidewater Cycle", then
"38,400 words · Complete · Updated Aug 14, 2026". The rating is the first of
its labels.

## About the work

| Field | What it is |
|---|---|
| `form` | `novel`, `novella`, `short-story` or `book`, shown as its label. Anything else is shown as written |
| `series` | `{"name": "...", "number": 2}`, shown as "Book 2 of ...". Each book in a series is its own card. Any card may have one, see [Fields every card has](README.md#fields-every-card-has) |
| `words` | The word count. Without it, the chapters' `words` are added up |
| `status` | `complete` or `in-progress` |
| `rating` | As you rate it: `General`, `Teen`, `Mature`, `Explicit`, or your own scale. Any card may have one |
| `warnings` | What a reader should know before starting |
| `fandoms` | For fan works: the works it's set in |
| `characters` | Who's in it |
| `languages` | Language codes: `["en"]`, `["de", "fr"]`. Shown by name. Any card may have them |

**Warnings** appear in a box under the title, before anything to read,
whatever the rating. Zephyr shows the rating and warnings as you give them,
and never hides a work because of them.

## The whole work

`files` offers the whole work in each format. **Read** opens the first that
Zephyr reads. When there are several, Read is a menu of them. **Download**
works the same way, with each format's size.

| Format | Read in Zephyr |
|---|---|
| EPUB (`.epub`) | Yes, in the book reader |
| Mobipocket and Kindle (`.mobi`, `.azw`, `.azw3`, `.prc`) | Yes, in the book reader |
| PDF (`.pdf`) | Yes, in the PDF viewer |
| Plain text (`.txt`) | Yes |
| HTML, RTF, Word (`.doc`, `.docx`), OpenDocument (`.odt`) | Download only |

Zephyr decides by the file's name, so name files with their usual extension.
**EPUB reads best:** its text reflows to the window, with chapters to jump
between.

## A chapter at a time

A work published in parts lists `chapters`, with or without `files` for the
whole:

```json
"status": "in-progress",
"chapters": [
  {"number": 1, "title": "The Lamp Room", "released": "2026-09-04", "words": 3120,
   "files": [{"format": "epub", "file": "chapters/01.epub", "sha256": "..."}]},
  {"number": 2, "title": "What the Tide Left", "released": "2026-09-18", "words": 2875,
   "files": [{"format": "epub", "file": "chapters/02.epub", "sha256": "..."}]}
]
```

Each chapter shows its number, title, date and words, with Read and Download
buttons. Double-click a chapter to read it, or right-click it for each format.

A work in progress reads "In progress · 2 chapters so far". The library
dates it by its **newest chapter**, so posting one brings the work to the
front. To post a chapter, add its file and a line to `chapters`.

## Filters for books

Library filters for books are:
- `vendor` (shown as Author), `fandom`, `rating`, `warning`, `status`, `form`;
- `words`, in bands: under 10k, 10k–40k, 40k–100k, 100k and over;
- `language`, `series`, `character`, `genre`, `tag`, `year` and `decade`.

See [Libraries](Libraries.md#filters). The search also finds chapter
titles, characters, fandoms, warnings and series names.
