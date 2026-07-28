# Guide: shape complex scripts

A plain per-rune advance loop (as in
[Render text to a PNG](render-text-to-png.md)) is enough for isolated Latin
text, but it breaks down for anything that needs **reordering** or
**cursive joining** — a right-to-left Arabic run, or Arabic letters that
must connect to their neighbours. That's what
[`shape`](../shape.md) is for.

## A right-to-left Arabic string

```go
package main

import (
	"fmt"

	"github.com/go-opentype/opentype"
	"github.com/go-opentype/shape"
)

func main() {
	f, err := opentype.Parse(arabicTTF) // a font with Arabic coverage, e.g. Noto Sans Arabic
	if err != nil {
		panic(err)
	}
	face := f.NewFace(32)

	glyphs := shape.Shape(face, "بيت", shape.Options{}) // "house"

	penX, penY := 0, 0
	for _, g := range glyphs {
		x := penX + g.XOffset
		y := penY + g.YOffset
		fmt.Printf("draw glyph %d at (%d,%d)\n", g.GID, x, y)
		penX += g.XAdvance
		penY += g.YAdvance
	}
}
```

`shape.Shape` already returns the glyphs in **visual order** — left to right
in drawing order — even though the source string is logically right-to-left.
You do not call [`bidi`](../bidi.md) yourself for this; `shape` composes it
internally.

## Mixed-direction text

For a paragraph that mixes scripts (English embedded in Arabic, or vice
versa), leave `Options.Direction` at its default `Auto`: the base direction
is picked from the first strong character per UAX #9 rules P2/P3, and each
directional run within the string is shaped and reordered independently,
then concatenated in visual order.

```go
glyphs := shape.Shape(face, "The word بيت means house", shape.Options{})
```

If you need the paragraph (or embedding) direction pinned rather than
auto-detected — e.g. a UI that always lays out RTL regardless of the first
character — set it explicitly:

```go
glyphs := shape.Shape(face, text, shape.Options{
    Direction: bidi.RightToLeft,
})
```

## Forcing a script

Script is auto-detected (any rune in the Arabic block selects the Arabic
shaper; everything else takes the Latin/default path). Override it with
`Options.Script` when you know better than the auto-detector — for example
a Latin-only UI string that happens to contain an Arabic punctuation mark
you don't want triggering cursive joining:

```go
glyphs := shape.Shape(face, text, shape.Options{Script: "latn"})
```

## Indic and Universal Shaping Engine scripts

Devanagari, Bengali, Tamil, and the rest of the ten dedicated-shaper Indic
scripts get the full HarfBuzz "indic" model: syllable splitting, base/reph
detection, the pre-base-matra and reph reordering passes, and the layered
GSUB/GPOS feature pipeline (including mark/mkmk/abvm/blwm attachment):

```go
glyphs := shape.Shape(face, "नमस्ते", shape.Options{}) // Devanagari "hello"
```

Thai, Lao, Khmer, Myanmar, Tibetan and the rest of the scripts without a
bespoke shaper go through the Universal Shaping Engine (USE), which
classifies each run into syllabic categories, splits it into clusters,
reorders pre-base vowels/modifiers and repha, and runs the USE GSUB/GPOS
pipeline (with sakot/halant joining, split-vowel decomposition, and
dotted-circle insertion for defective clusters) — no code changes needed
beyond calling `shape.Shape`, since script detection is automatic. See
[shape's scope](../shape.md#scope) for the full list of scripts each model
covers.

## Vertical (CJK tategaki) and other scripts

Egyptian Hieroglyph quadrats and Hangul jamo composition are also handled
automatically from the script of the input text. For vertical writing mode
(CJK tategaki — top-to-bottom, `vert`/`vrt2` glyph forms), set
`Options.Vertical`:

```go
glyphs := shape.Shape(face, "こんにちは", shape.Options{Vertical: true})
```

`shape.Shape` is a complete solution across all of these paths — Arabic,
Indic, USE, Egyptian Hieroglyphs, Hangul, vertical, and Latin/default
(which also covers Cyrillic, Greek, and horizontal CJK).
