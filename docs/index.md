# go-opentype documentation

**A pure-Go, zero-dependency TrueType/OpenType text stack.** Four small
modules that together read a font file, resolve Unicode bidirectional and
complex-script text, and rasterise the result — with no `golang.org/x/*`
packages and no cgo anywhere in the stack.

!!! success "Zero third-party dependencies"
    Every module in the stack imports only the Go standard library
    (`image`, `encoding/binary`, `math`, `errors`, …). That is what lets the
    whole stack cross-compile trivially to any Go target, including
    `GOOS=js GOARCH=wasm`, with no C toolchain and no vendored assets.

## The modules

| Repo | Module path | What it is |
| --- | --- | --- |
| [`opentype`](opentype.md) | `github.com/go-opentype/opentype` | the engine — sfnt parsing + anti-aliased rasterisation |
| [`bidi`](bidi.md) | `github.com/go-opentype/bidi` | the Unicode Bidirectional Algorithm (UAX #9) |
| [`shape`](shape.md) | `github.com/go-opentype/shape` | a HarfBuzz-lite complex-text shaper composing the two above |
| [`fonts`](fonts.md) | `github.com/go-opentype/fonts` | 46 bundled, legible, per-family `go:embed`ded font families |
| [`docs`](https://github.com/go-opentype/docs) | — | this documentation site (MkDocs Material, versioned with mike) |
| [`brand`](https://github.com/go-opentype/brand) | — | logo and brand assets |

Each module is standalone and independently importable — `shape` is the only
one that depends on its two siblings; `opentype`, `bidi`, and `fonts` have no
intra-org dependencies of their own (`fonts` depends only on `opentype`'s
`*Font` type for its `Parse` convenience wrapper).

## What it replaces

`go-opentype` exists to functionally replace, for a Go program that blits
glyphs into a pixel buffer:

- the narrow slice of `golang.org/x/image/font` and `font/opentype` a
  glyph-blitting UI needs (parse a font, build a sized face, pull advances
  and coverage masks), and
- `golang.org/x/text/unicode/bidi` for laying out mixed left-to-right /
  right-to-left text.

The stack targets functional parity with the HarfBuzz + FreeType pair, and on
the paths it covers, it's there today: TrueType `glyf` **and** CFF/CFF2
outline parsing and rasterisation, every GSUB/GPOS lookup type plus GDEF
mark filtering, OpenType Variations (`fvar`/`avar`/`gvar`/`HVAR`/`VVAR`/
`MVAR`), a TrueType instruction hinter and a CFF/Type 2 stem grid-fitter,
vertical metrics, the OpenType `MATH` table for math-typesetting metrics
(mirroring HarfBuzz's `hb-ot-math`), the full UAX #9 bidi algorithm through
rule L2, and
complex-text shaping for Arabic, the Indic scripts, the Universal Shaping
Engine (Thai, Lao, Khmer, Myanmar, Tibetan and more), Egyptian Hieroglyph
quadrats, Hangul, vertical (CJK tategaki) layout, and Latin/default — see
each module's page for its exact support matrix.

## Install

```sh
go get github.com/go-opentype/opentype
go get github.com/go-opentype/bidi
go get github.com/go-opentype/shape
go get github.com/go-opentype/fonts
```

## Where to go next

- [opentype](opentype.md) — the sfnt parser and rasteriser: fonts, `cmap`,
  TrueType `glyf` and CFF/CFF2 outlines, variable fonts, GSUB/GPOS shaping,
  hinting, vertical metrics, the `MATH` table for math typesetting, font
  subsetting and descriptor accessors, and the full support matrix.
- [bidi](bidi.md) — the UAX #9 engine and what it implements vs. defers to a
  shaper.
- [shape](shape.md) — the complex-text shaper: Arabic cursive joining, Indic
  reordering, the Universal Shaping Engine, Egyptian Hieroglyph quadrats,
  Hangul, vertical layout, ligatures, mark attachment, kerning, and which
  scripts it covers.
- [fonts](fonts.md) — the 46 bundled families and the per-family import
  model to reach them.
- **Guides** — task-oriented walkthroughs: rendering text to a PNG, shaping a
  mixed-direction Arabic string, and picking a bundled font for legibility.

Source lives at [github.com/go-opentype](https://github.com/go-opentype).
