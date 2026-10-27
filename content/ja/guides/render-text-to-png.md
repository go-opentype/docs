---
title: "テキストを PNG に描画する"
weight: 70
---

# ガイド: テキストを PNG に描画する {#guide-render-text-to-a-png}

ここでは、実用的な最小のパイプラインを順に説明します。同梱フォントを解析し、文字列を 1 グリフ（glyph）ずつ `image.RGBA` にラスタライズ（rasterise）して、PNG として書き出します。使うのは [`opentype`]({{< relref "../opentype.md" >}}) と [`fonts`]({{< relref "../fonts.md" >}}) だけで、通常のラテン文字の文字列ならシェーピングは不要です。

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

ポイントは次のとおりです。

- `face.Metrics().Ascent` は、行の上端からベースラインまでのオフセットを返します。これに上マージンを足すと `baselineY` になります。
- `face.GlyphMask` は、見える描画のないグリフ（スペースなど）に対しては `ok == false` を返します。その場合は描画の呼び出しを省略しますが、ペンは進めてください。
- `advance` は、フェイスに選んだサイズですでにピクセル単位になっています。したがって、これを `penX` に加算していくだけで、通常のラテン文字の連なりに必要なレイアウトはすべてまかなえます。

孤立したラテン文字やデフォルトの連なり以外、つまりアラビア語、方向が混在する段落、リガチャ（ligature）、マークの配置などを扱う場合は、上のルーン単位のループを [`shape.Shape`]({{< relref "../shape.md" >}}) に置き換えてください。`XOffset`/`YOffset`/`XAdvance`/`YAdvance` がすでに解決された `[]Glyph` が返ります。詳しくは [複雑なスクリプトをシェーピングする]({{< relref "shape-complex-scripts.md" >}}) を参照してください。
