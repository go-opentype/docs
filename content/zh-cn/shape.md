---
title: "shape"
weight: 40
---

# shape：复杂文本整形器 {#shape-the-complex-text-shaper}

`github.com/go-opentype/shape` 是纯 Go 的复杂文本整形器（shaper），使用 `CGO_ENABLED=0` 构建，**仅依赖标准库**，可以说是 go-opentype 栈里的轻量版 HarfBuzz。它把一段 Unicode 文本转换成按**视觉顺序**排列、已定位的字形（glyph），使阿拉伯文、印度系文字、东南亚文字、CJK 竖排和埃及圣书体文本能够正确渲染，而不是变成彼此孤立、没有附着、没有重排的字形。

它只组合两个同级模块，不添加其他内容：来自 [`bidi`]({{< relref "bidi.md" >}}) 的双向重排，以及来自 [`opentype`]({{< relref "opentype.md" >}}) 的由 GSUB/GPOS 驱动的替换与定位。没有 `golang.org/x/*`，没有第三方模块；它可以为所有 Go 目标平台构建，包括 `GOOS=js GOARCH=wasm`。

## 它能做什么 {#what-it-does}

- **双向重排**：解析 UAX #9 嵌入层级（通过 [`bidi`]({{< relref "bidi.md" >}})），并将字形从左到右排布，使右到左的片段按绘制顺序输出。
- **阿拉伯文连写**：解析每个字母的连接形式（独立/词首/词中/词尾），并**按位置**应用字体的 `isol`/`init`/`medi`/`fina` GSUB 特性，每个特性只作用于处于该形式的字形。连接形式会在 `ccmp` 分解过程中被跟踪（这是 Noto Sans Arabic 等真实字体所采用的“骨架加点”架构），因此连写后的字形确实会相连。
- **印度系文字整形**：天城文、孟加拉文、古木基文、古吉拉特文、奥里亚文、泰米尔文、泰卢固文、卡纳达文、马拉雅拉姆文和僧伽罗文，采用 HarfBuzz 的“indic”模型：音节拆分、基字/reph 检测、两遍重排（前置元音符号 matra 和 reph）、完整的基本与表现形式 GSUB 特性流水线，以及 GPOS 的 mark/mkmk/abvm/blwm 附着。
- **通用整形引擎（Universal Shaping Engine，USE）**：适用于没有专用整形器的文字的通用复杂文字模型：泰文、老挝文、高棉文、缅甸文、藏文、爪哇文、巴厘文、布吉文、老傣文等。片段被归入 USE 音节类别，拆分成簇（cluster），重排（前置元音和修饰符置于基字之前，repha 置于其后），并经过 USE 的 GSUB/GPOS 流水线，同时处理 sakot/halant 连接、分裂元音分解，以及为残缺簇插入虚线圆圈。
- **埃及圣书体 quadrat**：Unicode 格式控制字符 U+13430–U+1345F（连接符、角插入、叠加、分段/围框分隔符）被解析成二维 quadrat 树并按几何方式排布，使一串符号渲染成紧凑的方块。
- **谚文**：将字母（jamo）组合成预组合的音节块。
- **竖排书写模式（CJK tategaki）**：`Options.Vertical` 选用 `vert`/`vrt2` 直立字形，并使用字体的垂直度量（`vmtx`/`VORG`）自上而下堆叠字形。
- **连字、标记附着、字距调整**：对每种文字路径，先应用 GSUB 的 `ccmp`/`rlig`/`liga`/`calt`，再应用 GPOS 的 `kern`/`mark`/`mkmk`/`curs`，使变音符号落在其基字上，字对得到字距调整。

## 用法 {#usage}

```go
import (
    "github.com/go-opentype/opentype"
    "github.com/go-opentype/shape"
)

f, _ := opentype.Parse(ttf) // ttf is a []byte TrueType/OpenType blob
face := f.NewFace(32)        // 32px per em

for _, g := range shape.Shape(face, "بيت", shape.Options{}) {
    // g.GID      glyph to draw
    // g.Cluster  source rune index it derives from
    // g.XOffset, g.YOffset   placement relative to the pen (px)
    // g.XAdvance, g.YAdvance advance to move the pen by (px)
    // g.Scale    draw scale (< 1 inside an Egyptian quadrat)
}
```

基础方向默认为 `Auto`（取自第一个强方向字符），文字则从文本中自动检测（任何阿拉伯文区块的 rune 都会选用阿拉伯文，印度系/USE 的 rune 选用各自的整形器，等等），除非你设置了 `Options.Script` 或 `Options.Direction`。

## API {#api}

```go
type Glyph struct {
    GID      opentype.GlyphIndex // glyph to draw
    Cluster  int                 // source rune index
    XAdvance int
    YAdvance int
    XOffset  int
    YOffset  int
    Scale    float64 // 1.0 normally; < 1 inside an Egyptian quadrat
}

type Options struct {
    Direction bidi.Direction // LeftToRight, RightToLeft, Auto (default)
    Script    string         // "arab", "latn", "dflt", a script tag, or "" to auto-detect
    Features  []string       // extra feature tags applied over the whole run
    Vertical  bool           // CJK tategaki: top-to-bottom, vert/vrt2 forms
}

func Shape(face *opentype.Face, text string, opts Options) []Glyph
```

## 范围 {#scope}

**已实现：**阿拉伯文，十种拥有专用整形器的**印度系**文字，**通用整形引擎**（泰文、老挝文、高棉文、缅甸文、藏文、爪哇文、巴厘文、布吉文、老傣文等），**埃及圣书体** quadrat 排布，**谚文**字母组合，**竖排**书写模式，以及**拉丁文/默认**（拉丁文、西里尔文、希腊文、横排 CJK 等）整形。

对于一对一替换（阿拉伯文的位置形式），簇索引是精确的；当替换改变了片段长度时（连字、分解、印度系/USE 重排），簇索引是尽力而为的，并保持单调。

源代码：[github.com/go-opentype/shape](https://github.com/go-opentype/shape) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/shape)
