---
title: "整形复杂文字"
weight: 80
---

# 指南：整形复杂文字 {#guide-shape-complex-scripts}

简单的逐 rune 步进循环（如[将文本渲染为 PNG]({{< relref "render-text-to-png.md" >}})中所示）足以处理孤立的拉丁文文本，但对于任何需要**重排**或**连写**的内容就会失效，例如右到左的阿拉伯文片段，或必须与相邻字母相连的阿拉伯文字母。这正是 [`shape`]({{< relref "../shape.md" >}}) 的用途。

## 一个右到左的阿拉伯文字符串 {#a-right-to-left-arabic-string}

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

尽管源字符串在逻辑上是右到左的，`shape.Shape` 返回的字形（glyph）已经是**视觉顺序**，即按绘制顺序从左到右。为此你不必自己调用 [`bidi`]({{< relref "../bidi.md" >}})；`shape` 在内部已经组合了它。

## 混合方向的文本 {#mixed-direction-text}

对于混合多种文字的段落（阿拉伯文中嵌入英文，或反之），让 `Options.Direction` 保持默认的 `Auto`：基础方向按 UAX #9 规则 P2/P3 取自第一个强方向字符，字符串内的每个方向片段各自独立地整形与重排，然后按视觉顺序拼接。

```go
glyphs := shape.Shape(face, "The word بيت means house", shape.Options{})
```

如果你需要固定段落（或嵌入）的方向，而不是自动检测，例如无论第一个字符是什么，界面总是按 RTL 排布，可以显式设置：

```go
glyphs := shape.Shape(face, text, shape.Options{
    Direction: bidi.RightToLeft,
})
```

## 强制指定文字 {#forcing-a-script}

文字（script）是自动检测的（阿拉伯文区块中的任何 rune 都会选用阿拉伯文整形器，其余内容走拉丁文/默认路径）。当你比自动检测器更清楚时，可以用 `Options.Script` 覆盖，例如一个纯拉丁文的界面字符串碰巧包含一个阿拉伯文标点，而你不希望它触发连写：

```go
glyphs := shape.Shape(face, text, shape.Options{Script: "latn"})
```

## 印度系文字与通用整形引擎文字 {#indic-and-universal-shaping-engine-scripts}

天城文、孟加拉文、泰米尔文以及其余拥有专用整形器的十种印度系文字，采用完整的 HarfBuzz“indic”模型：音节拆分、基字/reph 检测、前置元音符号和 reph 的重排，以及分层的 GSUB/GPOS 特性流水线（包括 mark/mkmk/abvm/blwm 附着）：

```go
glyphs := shape.Shape(face, "नमस्ते", shape.Options{}) // Devanagari "hello"
```

泰文、老挝文、高棉文、缅甸文、藏文以及其余没有专用整形器的文字，则走通用整形引擎（Universal Shaping Engine，USE）：它把每个片段归入音节类别，拆分成簇，重排前置元音/修饰符和 repha，并运行 USE 的 GSUB/GPOS 流水线（包括 sakot/halant 连接、分裂元音分解，以及为残缺簇插入虚线圆圈）。由于文字检测是自动的，除了调用 `shape.Shape` 之外，无需修改任何代码。每种模型所覆盖的完整文字列表，请参见 [shape 的范围]({{< relref "../shape.md#scope" >}})。

## 竖排（CJK tategaki）及其他文字 {#vertical-cjk-tategaki-and-other-scripts}

埃及圣书体 quadrat 和谚文字母组合，同样会根据输入文本的文字自动处理。对于竖排书写模式（CJK tategaki，自上而下，使用 `vert`/`vrt2` 字形），请设置 `Options.Vertical`：

```go
glyphs := shape.Shape(face, "こんにちは", shape.Options{Vertical: true})
```

`shape.Shape` 是覆盖上述所有路径的完整方案：阿拉伯文、印度系文字、USE、埃及圣书体、谚文、竖排，以及拉丁文/默认（同时涵盖西里尔文、希腊文和横排 CJK）。
