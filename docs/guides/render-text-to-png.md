# Guide: render text to a PNG

This walks through the smallest useful pipeline: parse a bundled font,
rasterise a string glyph-by-glyph into an `image.RGBA`, and write it out as a
PNG. It only touches [`opentype`](../opentype.md) and
[`fonts`](../fonts.md) — no shaping needed for a plain Latin string.

```go
package main

import (
	"image"
	"image/color"
	"image/draw"
	"image/png"
	"os"

	"github.com/go-opentype/fonts"
	"github.com/go-opentype/opentype"
)

func main() {
	f, err := opentype.Parse(fonts.MostLegible()) // Atkinson Hyperlegible
	if err != nil {
		panic(err)
	}
	face := f.NewFace(32) // 32px per em

	text := "Hello, go-opentype"
	w := face.Measure(text)
	m := face.Metrics()
	h := m.Height

	dst := image.NewRGBA(image.Rect(0, 0, w+8, h+8))
	draw.Draw(dst, dst.Bounds(), &image.Uniform{C: color.White}, image.Point{}, draw.Src)

	penX := 4
	baselineY := 4 + m.Ascent
	ink := image.NewUniform(color.Black)

	for _, r := range text {
		bounds, mask, maskp, advance, ok := face.GlyphMask(r, penX, baselineY)
		if ok && mask != nil {
			draw.DrawMask(dst, bounds, ink, image.Point{}, mask, maskp, draw.Over)
		}
		penX += advance
	}

	out, err := os.Create("hello.png")
	if err != nil {
		panic(err)
	}
	defer out.Close()
	if err := png.Encode(out, dst); err != nil {
		panic(err)
	}
}
```

Key points:

- `face.Metrics().Ascent` gives you the baseline offset from the top of the
  line — add it to your top margin to get `baselineY`.
- `face.GlyphMask` returns `ok == false` for glyphs with no visible ink
  (like a space); skip the draw call but still advance the pen.
- `advance` is already in pixels at the face's chosen size, so accumulating
  it into `penX` is all the layout a plain Latin run needs.

For anything beyond isolated Latin/default runs — Arabic, mixed-direction
paragraphs, ligatures, mark attachment — swap the per-rune loop above for
[`shape.Shape`](../shape.md), which returns a `[]Glyph` with `XOffset`/
`YOffset`/`XAdvance`/`YAdvance` already resolved; see
[Shape complex scripts](shape-complex-scripts.md).
