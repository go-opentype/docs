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

## Mathematical typesetting (the `MATH` table)

`opentype` decodes the OpenType **`MATH`** table: the font-level metrics a
math-typesetting engine (LuaTeX, unicode-math, MathJax, Word's equation
editor, or anything built on HarfBuzz's `hb-ot-math`) needs to lay out
formulas. It has three parts, all exposed pixel-scaled through a `Face`:

- **MathConstants** — around 56 global scalars: the math axis height,
  subscript/superscript shift and gap minimums, fraction and radical gaps,
  rule thicknesses, and more, read with `MathConstant` and a `MathConstant`
  value such as `opentype.AxisHeight` or `opentype.FractionRuleThickness`
  (see `go doc` for the full, on-disk-ordered list).
- **MathGlyphInfo** — per-glyph *italic correction* (`ItalicCorrection`, the
  space to insert after a slanted glyph before an upright one, or the
  horizontal offset for a subscript that follows a superscript), *top-accent
  attachment* (`TopAccentAttachment`, the x position at which to centre an
  accent over a glyph), an *extended-shape* flag (`IsExtendedShapeGlyph`, for
  glyphs such as a large integral whose superscript is positioned as if the
  glyph were stretched), and four-corner cut-in *math kerning*
  (`MathKern`, a step function of the adjacent script's attachment height).
- **MathVariants** — larger size variants of a glyph and, when even the
  largest variant isn't big enough, the recipe for assembling an arbitrarily
  tall or wide stretchy glyph (integrals, braces, radicals, arrows) out of
  repeatable parts, both returned together by `MathVariants`.

Math *layout* itself — the box model, TeX's math-list rules, choosing a
variant or growing an assembly to a target size — is a higher-level concern
built on top of these metrics; it is out of scope for this package, exactly
as `hb-ot-math` only exposes the metrics and leaves layout to the caller.

```go
f, err := opentype.Parse(stixTwoMathOTF) // an OpenType math font, e.g. STIX Two Math
if err != nil {
    log.Fatal(err)
}
if !f.HasMath() {
    log.Fatal("font carries no MATH table")
}
face := f.NewFace(32) // 32px per em

axisHeight := face.MathConstant(opentype.AxisHeight)
ruleThickness := face.MathConstant(opentype.FractionRuleThickness)
fmt.Println("axis height:", axisHeight, "fraction rule thickness:", ruleThickness)

// Math-italic glyphs (the U+1D400-U+1D7FF alphanumeric symbols block) commonly
// carry a nonzero italic correction.
mathItalicF, _ := f.GlyphIndex('𝑓') // MATHEMATICAL ITALIC SMALL F, U+1D453
fmt.Println("italic correction:", face.ItalicCorrection(mathItalicF))

// A classic stretchy delimiter exposes both size variants and a glyph
// assembly a layout engine can grow to any height.
paren, _ := f.GlyphIndex('(')
variants, asm := face.MathVariants(paren, true) // true = vertical axis
fmt.Println("size variants:", len(variants), "assembly parts:", len(asm.Parts))
```

Run against the real STIX Two Math font at 32px per em, this prints `axis
height: 8 fraction rule thickness: 2`, `italic correction: 1`, and `size
variants: 13 assembly parts: 3`. STIX Two Math (SIL Open Font License) is
bundled as [`testdata/STIXTwoMath-Regular.otf`](https://github.com/go-opentype/opentype/blob/main/testdata/STIXTwoMath-Regular.otf)
in the `opentype` repository and is checked end-to-end in CI, alongside the
synthetic MATH-table fixtures that give the decoder 100% branch coverage.

## Font descriptors and subsetting

For a program that *embeds* fonts — a PDF writer, a CSS `@font-face`
generator — `opentype` exposes the writer-side primitives too:

- **Descriptor scalars** from OS/2 and `head`/`post`, ready to fill a PDF
  `/FontDescriptor` or a CSS declaration: `Ascent`, `Descent`, `LineGap`,
  `FontBBox`, `CapHeight`, `XHeight`, `ItalicAngle`, `WeightClass`,
  `WidthClass`, `StemV`, `Flags`, and the `IsFixedPitch`/`IsItalic`/`IsSerif`
  booleans.
- **Instancing** — `Instance(coords)` returns a new `*Font` baked to a single
  point in a variable font's design space (glyf/CFF2 outlines and metrics
  frozen), and `InstanceBytes(coords)` returns that instance as a fresh sfnt
  blob.
- **Subsetting** — `SubsetTrueType(gids)` cuts a TrueType font down to the
  requested glyphs, closing composite-glyph dependencies and returning the
  old→new glyph-id remap; `SubsetCFF(gids)` does the same for CFF/OTTO fonts,
  closing `seac` accent references. Both keep only the glyphs a document
  actually uses, so embedded fonts stay small.

```go
f, _ := opentype.Parse(ttf)

// Descriptor scalars for a PDF /FontDescriptor.
xMin, yMin, xMax, yMax := f.FontBBox()
fmt.Println(f.Ascent(), f.Descent(), f.CapHeight(), f.StemV(), xMin, yMin, xMax, yMax)

// Subset to the glyphs a document uses (here: 'H' and 'i').
h, _ := f.GlyphIndex('H')
i, _ := f.GlyphIndex('i')
sub, oldToNew, _ := f.SubsetTrueType([]opentype.GlyphIndex{h, i})
fmt.Println(len(sub), oldToNew)
```

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
| `(*Font).HasMath() bool` | Whether the font carries an OpenType `MATH` table. |
| `(*Font).IsExtendedShapeGlyph(gid GlyphIndex) bool` | Whether gid is in the `MATH` extended-shape coverage. |
| `(*Face).MathConstant(which MathConstant) int` | One of ~56 pixel-scaled `MATH` layout constants (axis height, script shifts, fraction/radical gaps, …). |
| `(*Face).ItalicCorrection(gid GlyphIndex) int` | Per-glyph math italic correction, in pixels. |
| `(*Face).TopAccentAttachment(gid GlyphIndex) (int, bool)` | X position (pixels) to centre a top accent over gid. |
| `(*Face).MathKern(gid GlyphIndex, corner MathKernCorner, correctionHeight int) int` | Cut-in math kern (pixels) for one of gid's four corners at a given attachment height. |
| `(*Face).MathVariants(gid GlyphIndex, vertical bool) ([]MathVariant, *MathAssembly)` | Size variants plus stretchy-glyph assembly for gid along one axis. |
| `(*Font).Ascent/Descent/LineGap/FontBBox/CapHeight/XHeight/ItalicAngle/WeightClass/WidthClass/StemV/Flags/IsFixedPitch/IsItalic/IsSerif` | Font-descriptor scalars from OS/2 + `head`/`post` for PDF `/FontDescriptor` and CSS `@font-face`. |
| `(*Font).Instance(coords map[string]float64) (*Font, error)` / `(*Font).InstanceBytes(coords) ([]byte, error)` | Bake a variable font to a static instance (`*Font` or a new sfnt blob). |
| `(*Font).SubsetTrueType(gids []GlyphIndex) ([]byte, map[GlyphIndex]GlyphIndex, error)` | Subset a TrueType font to a glyph set, closing composites; returns the old→new id map. |
| `(*Font).SubsetCFF(gids []GlyphIndex) ([]byte, error)` | Subset a CFF/OTTO font to a glyph set, closing `seac` accents. |

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
- the OpenType **`MATH`** table — math-typesetting metrics (constants,
  per-glyph italic correction / top-accent attachment / four-corner kerning,
  and stretchy-glyph size variants and assemblies), exposed pixel-scaled
  through a `Face` (`HasMath`, `MathConstant`, `ItalicCorrection`,
  `TopAccentAttachment`, `MathKern`, `MathVariants`,
  `IsExtendedShapeGlyph`); math *layout* is left to a higher-level engine
- OS/2 + `head`/`post` **font-descriptor** scalars (`Ascent`, `Descent`,
  `FontBBox`, `CapHeight`, `XHeight`, `ItalicAngle`, `WeightClass`,
  `WidthClass`, `StemV`, `Flags`, …) for PDF and CSS embedding
- **instancing** a variable font to a static master (`Instance`,
  `InstanceBytes`) and **subsetting** to a glyph set (`SubsetTrueType` with
  composite closure and an old→new id map, `SubsetCFF` with `seac` closure)
- anti-aliased rasterisation via 4×4 supersampling under the non-zero
  winding rule

Rasterisation uses uniform supersampling (not the delta-hinted rasterisation
of a native TrueType/CFF hinter's drop-out control), so it favours
correctness and portability over the very last drop of small-size sharpness.
This is a feature-complete engine on the paths HarfBuzz + FreeType cover —
nothing above is a stub.

## Testing

Tests never depend on an external font to reach 100% coverage: they
synthesise minimal-but-valid TrueType and CFF fonts in memory (table
directory + `head`/`maxp`/`hhea`/`hmtx`/`cmap`/`loca`/`glyf`, or a synthetic
`MATH` table) to deterministically exercise every parse and raster branch,
including the error paths. Real-world fonts — Adobe's Source Serif 4 and,
for the `MATH` table, STIX Two Math (both SIL Open Font License, bundled
under `testdata/`) — are additionally exercised end-to-end in
`example_test.go` and a handful of sanity-check tests, so the documented
examples double as smoke tests against production font data. CI enforces
**100.0% statement coverage**, `go vet`, and a cross-compile smoke over
`linux/{amd64,arm64,riscv64,loong64,ppc64le,s390x}`, `js/wasm`,
`darwin/arm64` and `windows/amd64`.

Source: [github.com/go-opentype/opentype](https://github.com/go-opentype/opentype) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/opentype)
