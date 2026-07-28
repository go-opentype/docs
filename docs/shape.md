# shape — the complex-text shaper

`github.com/go-opentype/shape` is a pure-Go, `CGO_ENABLED=0`,
**standard-library-only** complex-text shaper — a HarfBuzz-lite for the
go-opentype stack. It turns a run of Unicode text into positioned glyphs in
**visual order**, so Arabic, Indic, Southeast-Asian, CJK vertical and
Egyptian Hieroglyph text renders correctly instead of as isolated,
unattached, unreordered glyphs.

It composes two siblings and adds nothing else: bidi reordering from
[`bidi`](bidi.md) and GSUB/GPOS-driven substitution/positioning from
[`opentype`](opentype.md). No `golang.org/x/*`, no third-party modules; it
builds for every Go target including `GOOS=js GOARCH=wasm`.

## What it does

- **Bidirectional reordering** — resolves UAX #9 embedding levels (via
  [`bidi`](bidi.md)) and lays the glyphs out left-to-right, so a
  right-to-left run is emitted in drawing order.
- **Arabic cursive joining** — resolves each letter's joining form
  (isolated / initial / medial / final) and applies the font's
  `isol`/`init`/`medi`/`fina` GSUB features **positionally**, each only at
  the glyphs in that form. Joining forms are tracked through `ccmp`
  decomposition (the rasm-skeleton-plus-dots architecture real fonts such as
  Noto Sans Arabic use), so the joined glyphs actually connect.
- **Indic shaping** — Devanagari, Bengali, Gurmukhi, Gujarati, Oriya, Tamil,
  Telugu, Kannada, Malayalam and Sinhala, via the HarfBuzz "indic" model:
  syllable splitting, base/reph detection, two reordering passes (pre-base
  matras and reph), the full basic + presentation GSUB feature pipeline, and
  GPOS mark/mkmk/abvm/blwm attachment.
- **Universal Shaping Engine (USE)** — the general complex-script model for
  scripts without a bespoke shaper: Thai, Lao, Khmer, Myanmar, Tibetan,
  Javanese, Balinese, Buginese, Tai Tham and more. Runs are classified into
  USE syllabic categories, split into clusters, reordered (pre-base vowels
  and modifiers before the base, repha after it) and run through the USE
  GSUB/GPOS pipeline, with sakot/halant joining, split-vowel decomposition
  and dotted-circle insertion for defective clusters.
- **Egyptian Hieroglyph quadrats** — the Unicode format-control characters
  U+13430–U+1345F (joiners, corner insertions, overlays, segment/enclosure
  delimiters) are parsed into a two-dimensional quadrat tree and laid out
  geometrically, so a run of signs renders as compact blocks.
- **Hangul** — jamo composition into precomposed syllable blocks.
- **Vertical writing mode (CJK tategaki)** — `Options.Vertical` selects the
  `vert`/`vrt2` upright glyph forms and stacks glyphs top-to-bottom using the
  font's vertical metrics (`vmtx`/`VORG`).
- **Ligatures, mark attachment, kerning** — GSUB `ccmp`/`rlig`/`liga`/`calt`
  then GPOS `kern`/`mark`/`mkmk`/`curs` for every script path, so diacritics
  sit on their base and pairs kern.

## Usage

```go
import (
    "github.com/go-opentype/opentype"
    "github.com/go-opentype/shape"
)

f, _ := opentype.Parse(ttf) // ttf is a []byte TrueType/OpenType blob
face := f.NewFace(32)        // 32px per em

for _, g := range shape.Shape(face, "بيت", shape.Options{}) {
    // g.GID      glyph to draw
    // g.Cluster  source rune index it derives from
    // g.XOffset, g.YOffset   placement relative to the pen (px)
    // g.XAdvance, g.YAdvance advance to move the pen by (px)
    // g.Scale    draw scale (< 1 inside an Egyptian quadrat)
}
```

The base direction defaults to `Auto` (from the first strong character) and
the script is auto-detected from the text (any Arabic-block rune selects
Arabic, Indic/USE runes select their shaper, ...) unless you set
`Options.Script` or `Options.Direction`.

## API

```go
type Glyph struct {
    GID      opentype.GlyphIndex // glyph to draw
    Cluster  int                 // source rune index
    XAdvance int
    YAdvance int
    XOffset  int
    YOffset  int
    Scale    float64 // 1.0 normally; < 1 inside an Egyptian quadrat
}

type Options struct {
    Direction bidi.Direction // LeftToRight, RightToLeft, Auto (default)
    Script    string         // "arab", "latn", "dflt", a script tag, or "" to auto-detect
    Features  []string       // extra feature tags applied over the whole run
    Vertical  bool           // CJK tategaki: top-to-bottom, vert/vrt2 forms
}

func Shape(face *opentype.Face, text string, opts Options) []Glyph
```

## Scope

**Implemented:** Arabic, the ten dedicated-shaper **Indic** scripts, the
**Universal Shaping Engine** (Thai, Lao, Khmer, Myanmar, Tibetan, Javanese,
Balinese, Buginese, Tai Tham, ...), **Egyptian Hieroglyph** quadrat layout,
**Hangul** jamo composition, **vertical** writing mode, and
**Latin/default** (Latin, Cyrillic, Greek, CJK horizontal, ...) shaping.

Cluster indices are exact for one-to-one substitutions (the Arabic
positional forms) and best-effort, monotonic, when a substitution changes
the run length (ligatures, decomposition, Indic/USE reordering).

Source: [github.com/go-opentype/shape](https://github.com/go-opentype/shape) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/shape)
