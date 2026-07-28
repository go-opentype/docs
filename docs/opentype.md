# opentype — the engine

`github.com/go-opentype/opentype` is a pure-Go, `CGO_ENABLED=0`,
**standard-library-only** parser and anti-aliased rasteriser for
TrueType/OpenType fonts. It imports only `image`, `encoding/binary`, `math`,
`errors` and friends — no `golang.org/x/*`, no third-party modules — so it
builds for every Go target including `GOOS=js GOARCH=wasm`.

It exists to functionally replace the narrow slice of
`golang.org/x/image/font/opentype` that a glyph-blitting UI needs — parse a
font blob, build a face at a pixel size, pull per-rune advances and 8-bit
alpha coverage masks — and then goes further: CFF/CFF2 outlines, variable
fonts, GPOS/GSUB shaping, kerning and TrueType/CFF hinting are all in scope.

## Usage

```go
ttf, err := os.ReadFile("MyFont.ttf") // or .otf (CFF/CFF2)
if err != nil {
    log.Fatal(err)
}
f, err := opentype.Parse(ttf)
if err != nil {
    log.Fatal(err)
}
face := f.NewFace(16) // 16px per em

m := face.Metrics()             // Metrics{Ascent, Descent, Height} in pixels
w := face.Measure("Hello")      // total advance width in pixels
adv := face.Advance('H')        // one rune's advance in pixels

// Rasterise a glyph with (penX, baselineY) as the pen origin on the baseline.
bounds, mask, maskp, advance, ok := face.GlyphMask('H', penX, baselineY)
if ok && mask != nil {
    // Composite mask (an *image.Alpha coverage mask) onto your destination:
    // pixel (maskp.X+i, maskp.Y+j) covers destination (bounds.Min.X+i, bounds.Min.Y+j).
}
```

The `GlyphMask` shape deliberately mirrors `x/image`'s `font.Face.Glyph`, so
swapping this in for the x/image face is mechanical.

## Variable fonts

Instance a variable font at a specific point in its design space with
`SetVariation`, keyed by axis tag (`wght`, `wdth`, `opsz`, ...):

```go
f, err := opentype.Parse(interVariableTTF) // a variable-font .ttf
if err != nil {
    log.Fatal(err)
}
face := f.NewFace(16)
face.SetVariation(map[string]float64{"wght": 600}) // Semibold instance

for _, ax := range f.Axes() {
    fmt.Println(ax.Tag, ax.Min, ax.Default, ax.Max)
}
for _, ni := range f.NamedInstances() {
    fmt.Println(ni.Name, ni.Coordinates)
}
```

`SetVariation` reinterpolates the glyph outlines (`gvar` for TrueType,
CFF2 blends for CFF2) and re-derives `HVAR`/`VVAR`/`MVAR`-driven metrics for
the chosen coordinates; `avar` axis-value mapping is applied automatically.

## Hinting

`SetHinting(true)` runs the TrueType instruction interpreter (for `glyf`
outlines) or the CFF/Type 2 stem grid-fitter (for CFF/CFF2 outlines) before
rasterisation; `SetStemDarkening(true)` additionally applies CFF stem
darkening. Both default to off (uniform 4×4-supersampled rendering).

## API

| Symbol | Purpose |
| --- | --- |
| `Parse(b []byte) (*Font, error)` | Decode an sfnt (TrueType or CFF/OTTO) font blob. |
| `(*Font).NumGlyphs() int` | Glyph count (from `maxp`). |
| `(*Font).GlyphIndex(r rune) (GlyphIndex, bool)` | Map a rune via the cmap. |
| `(*Font).GlyphIndexVariation(r, vs rune) (GlyphIndex, bool)` | Map a rune + Unicode variation selector. |
| `(*Font).Axes() []Axis` | Variable-font design axes (`fvar`). |
| `(*Font).NamedInstances() []NamedInstance` | Named positions in the variation space. |
| `(*Font).GPOS() *GPOS` / `(*Font).GSUB() *GSUB` | Parsed layout tables, or `nil`. |
| `(*Font).HasVerticalMetrics() bool` | Whether `vhea`/`vmtx` are present. |
| `(*Font).NewFace(sizePx int) *Face` | Build a sized face (scale = `sizePx / unitsPerEm`). |
| `(*Face).Metrics() Metrics` | `Ascent`, `Descent`, `Height` in pixels. |
| `(*Face).Advance(r rune) int` / `(*Face).Measure(s string) int` | Rune / string advance in pixels. |
| `(*Face).Kern(prev, r rune) int` / `(*Face).MeasureKerned(s string) int` | GPOS-or-`kern` pair adjustment. |
| `(*Face).GlyphMask(r, x, y) (image.Rectangle, *image.Alpha, image.Point, int, bool)` | Rasterised glyph + advance. |
| `(*Face).Shape(text string, features ...string) []GlyphIndex` | GSUB-substituted glyph run. |
| `(*Face).ShapePositioned(text string, features ...string) []PositionedGlyph` | GSUB + GPOS glyph run with pen offsets. |
| `(*Face).SetVariation(coords map[string]float64)` | Instance a variable font at the given axis coordinates. |
| `(*Face).SetHinting(on bool)` / `(*Face).SetStemDarkening(on bool)` | Toggle the TrueType/CFF hinter. |
| `(*Face).VerticalAdvance(r rune) int` / `(*Face).VerticalOrigin(r rune) (int, bool)` | Vertical writing-mode metrics. |

A `Font` is immutable after `Parse` and safe for concurrent use. A `Face`
caches rasterised glyphs and is **not** safe for concurrent use; build one
`Face` per goroutine if needed.

## Support matrix

- sfnt container (`0x00010000` and `true` TrueType magics, `OTTO` CFF magic)
- `head`, `maxp`, `hhea`, `hmtx` (including the trailing-run shared advance)
- `cmap` formats **0, 2, 4, 6, 8, 10, 12, 13** and **14** (Unicode variation
  sequences), preferring 12
- `loca` (short and long), `glyf` simple glyphs (repeat flags, short/long
  deltas) and composite glyphs (`ARGS_ARE_XY_VALUES`, scale / x&y-scale / 2×2,
  `MORE_COMPONENTS`, with cycle and depth guards)
- implied on-curve midpoint synthesis for quadratic contours
- CFF and CFF2 (`OTTO`) Type 2 charstring outlines, including subroutines,
  `seac` accent composition, and CFF2 variable-font blend operators
- `fvar` axes and named instances, `avar` axis-value mapping,
  `gvar`/CFF2 glyph-outline interpolation, `HVAR`/`VVAR`/`MVAR` metric
  variation, via `(*Face).SetVariation`
- every `GSUB`/`GPOS` lookup type — ligatures, contextual and positional-form
  substitution (`isol`/`init`/`medi`/`fina`), single/pair/cursive
  positioning, mark-to-base and mark-to-mark attachment — plus `GDEF` and
  lookup-flag mark filtering
- kerning via GPOS pair positioning with legacy `kern`-table fallback,
  through a single `Kerner`
- a TrueType instruction hinter and a CFF/Type 2 stem grid-fitter (blue
  zones, counters, stem darkening), toggled per `Face` with `SetHinting` /
  `SetStemDarkening`
- `vhea`/`vmtx`/`VORG`-aware vertical metrics and origins for vertical
  writing modes
- anti-aliased rasterisation via 4×4 supersampling under the non-zero
  winding rule

Rasterisation uses uniform supersampling (not the delta-hinted rasterisation
of a native TrueType/CFF hinter's drop-out control), so it favours
correctness and portability over the very last drop of small-size sharpness.
This is a feature-complete engine on the paths HarfBuzz + FreeType cover —
nothing above is a stub.

## Testing

Tests never depend on an external font: they synthesise minimal-but-valid
TrueType fonts in memory (table directory + `head`/`maxp`/`hhea`/`hmtx`/
`cmap`/`loca`/`glyf`) to deterministically exercise every parse and raster
branch, including the error paths. CI enforces **100.0% statement
coverage**, `go vet`, and a cross-compile smoke over
`linux/{amd64,arm64,riscv64,loong64,ppc64le,s390x}`, `js/wasm`,
`darwin/arm64` and `windows/amd64`.

Source: [github.com/go-opentype/opentype](https://github.com/go-opentype/opentype) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/opentype)
