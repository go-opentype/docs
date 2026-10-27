---
title: "将文本渲染为 PNG"
weight: 70
---

# 指南：将文本渲染为 PNG {#guide-render-text-to-a-png}

本指南演示最小而实用的流水线：解析一个内置字体（font），把字符串逐字形光栅化到 `image.RGBA` 中，再写出为 PNG。它只用到 [`opentype`]({{< relref "../opentype.md" >}}) 和 [`fonts`]({{< relref "../fonts.md" >}})，对于普通的拉丁文字符串，不需要文本整形（shaping）。

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

要点：

- `face.Metrics().Ascent` 给出基线相对于行顶的偏移，把它加到上边距上，就得到 `baselineY`。
- 对于没有可见笔画的字形（例如空格），`face.GlyphMask` 返回 `ok == false`；跳过绘制调用，但仍要推进笔位。
- `advance` 已经是该字面（face）所选大小下的像素值，因此把它累加到 `penX` 上，就是普通拉丁文片段所需的全部布局。

对于孤立拉丁文/默认文字片段之外的内容，例如阿拉伯文、混合方向的段落、连字、标记附着，请把上面逐 rune 的循环换成 [`shape.Shape`]({{< relref "../shape.md" >}})，它返回的 `[]Glyph` 已经解析好了 `XOffset`/`YOffset`/`XAdvance`/`YAdvance`；参见[整形复杂文字]({{< relref "shape-complex-scripts.md" >}})。
