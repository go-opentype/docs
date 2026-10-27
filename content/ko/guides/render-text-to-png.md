---
title: "텍스트를 PNG로 렌더링하기"
weight: 70
---

# 가이드: 텍스트를 PNG로 렌더링하기 {#guide-render-text-to-a-png}

가장 작지만 쓸모 있는 파이프라인을 따라가 봅니다. 내장 글꼴을 파싱하고, 문자열을 글리프(glyph) 하나씩 `image.RGBA`에 래스터화한 뒤, PNG로 저장합니다. [`opentype`]({{< relref "../opentype.md" >}})과 [`fonts`]({{< relref "../fonts.md" >}})만 사용하며, 일반적인 라틴 문자열에는 셰이핑(shaping)이 필요하지 않습니다.

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

핵심 사항:

- `face.Metrics().Ascent`는 줄 상단에서 베이스라인까지의 오프셋을 알려 줍니다. 여기에 위쪽 여백을 더하면 `baselineY`가 됩니다.
- `face.GlyphMask`는 보이는 잉크가 없는 글리프(공백 등)에 대해 `ok == false`를 반환합니다. 이 경우 그리기 호출은 건너뛰되 펜은 계속 전진시켜야 합니다.
- `advance`는 이미 페이스에 지정한 크기의 픽셀 단위이므로, 이를 `penX`에 누적하는 것이 일반적인 라틴 문자 구간에 필요한 레이아웃의 전부입니다.

고립된 라틴/기본 문자 구간을 넘어서는 경우, 즉 아랍어, 방향이 섞인 문단, 합자(ligature), 마크 부착 등에는 위의 룬별 루프를 [`shape.Shape`]({{< relref "../shape.md" >}})로 바꾸십시오. 이 함수는 `XOffset`/`YOffset`/`XAdvance`/`YAdvance`가 이미 결정된 `[]Glyph`를 반환합니다. [복합 문자 셰이핑하기]({{< relref "shape-complex-scripts.md" >}})를 참고하십시오.
