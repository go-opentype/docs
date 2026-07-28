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
| [`fonts`](fonts.md) | `github.com/go-opentype/fonts` | six bundled, legible, `go:embed`ded font families |
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

The long-term aim on the paths it covers is functional parity with the
HarfBuzz + FreeType pair. Today that means TrueType `glyf` outline parsing
and rasterisation, the full UAX #9 bidi algorithm through rule L2, and
Arabic + Latin/default complex-text shaping. **CFF/CFF2 outlines, full
GSUB/GPOS positioning, hinting, vertical metrics and OpenType Variations
(variable fonts) are tracked as roadmap** — see each module's page for its
exact support matrix; nothing here overstates what ships today.

## Install

```sh
go get github.com/go-opentype/opentype
go get github.com/go-opentype/bidi
go get github.com/go-opentype/shape
go get github.com/go-opentype/fonts
```

## Where to go next

- [opentype](opentype.md) — the sfnt parser and rasteriser: fonts, `cmap`,
  `glyf` outlines, and the current CFF/CFF2/GSUB/GPOS/hinting/vertical-metrics
  support matrix.
- [bidi](bidi.md) — the UAX #9 engine and what it implements vs. defers to a
  shaper.
- [shape](shape.md) — the complex-text shaper: Arabic cursive joining,
  ligatures, mark attachment, kerning, and which scripts it covers.
- [fonts](fonts.md) — the six bundled families and the two-line API to reach
  them.
- **Guides** — task-oriented walkthroughs: rendering text to a PNG, shaping a
  mixed-direction Arabic string, and picking a bundled font for legibility.

Source lives at [github.com/go-opentype](https://github.com/go-opentype).
