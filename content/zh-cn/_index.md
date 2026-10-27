---
title: "概览"
weight: 10
---

# go-opentype 文档 {#go-opentype-documentation}

**纯 Go、零依赖的 TrueType/OpenType 文本栈。** 四个小模块协同工作：读取字体文件，处理 Unicode 双向文本与复杂文字，并对结果进行光栅化（rasterise）——整个栈中没有 `golang.org/x/*` 包，也没有任何 cgo。

{{< callout type="info" >}}
**零第三方依赖**

栈中的每个模块都只导入 Go 标准库（`image`、`encoding/binary`、`math`、`errors` 等）。正因如此，整个栈可以轻松交叉编译到任何 Go 目标平台，包括 `GOOS=js GOARCH=wasm`，既不需要 C 工具链，也不需要附带第三方资源。
{{< /callout >}}

## 模块 {#the-modules}

| 仓库 | 模块路径 | 说明 |
| --- | --- | --- |
| [`opentype`]({{< relref "opentype.md" >}}) | `github.com/go-opentype/opentype` | 引擎：sfnt 解析与抗锯齿光栅化 |
| [`bidi`]({{< relref "bidi.md" >}}) | `github.com/go-opentype/bidi` | Unicode 双向算法（UAX #9） |
| [`shape`]({{< relref "shape.md" >}}) | `github.com/go-opentype/shape` | 轻量版 HarfBuzz 复杂文本整形器，由上述两个模块组合而成 |
| [`fonts`]({{< relref "fonts.md" >}}) | `github.com/go-opentype/fonts` | 46 个内置的、易读的字体族，每个字体族单独通过 `go:embed` 嵌入 |
| [`docs`](https://github.com/go-opentype/docs) | — | 本文档站点（MkDocs Material，使用 mike 做版本管理） |
| [`brand`](https://github.com/go-opentype/brand) | — | 徽标与品牌素材 |

每个模块都是独立的，可以单独导入：只有 `shape` 依赖它的两个同级模块；`opentype`、`bidi` 和 `fonts` 自身没有组织内部的依赖（`fonts` 仅为其便捷封装 `Parse` 而依赖 `opentype` 的 `*Font` 类型）。

## 它取代什么 {#what-it-replaces}

对于把字形（glyph）绘制到像素缓冲区的 Go 程序，`go-opentype` 的目标是在功能上取代：

- `golang.org/x/image/font` 和 `font/opentype` 中，字形绘制型界面所需的那一小部分（解析字体、构建指定大小的字面（face）、获取字形的步进宽度和覆盖蒙版），以及
- 用于排布左到右/右到左混合文本的 `golang.org/x/text/unicode/bidi`。

该栈的目标是在功能上与 HarfBuzz + FreeType 这一组合对等，并且在它所覆盖的路径上，如今已经做到：TrueType `glyf` **以及** CFF/CFF2 轮廓的解析与光栅化，所有 GSUB/GPOS 查找类型以及 GDEF 标记过滤，OpenType 可变字体（`fvar`/`avar`/`gvar`/`HVAR`/`VVAR`/`MVAR`），TrueType 指令字体提示器（hinter）和 CFF/Type 2 笔画网格适配器，垂直度量，用于数学排版度量的 OpenType `MATH` 表（对应 HarfBuzz 的 `hb-ot-math`），直到规则 L2 的完整 UAX #9 双向算法，以及针对阿拉伯文、印度系文字、通用整形引擎（Universal Shaping Engine，涵盖泰文、老挝文、高棉文、缅甸文、藏文等）、埃及圣书体 quadrat、谚文、竖排（CJK tategaki）排版以及拉丁文/默认文字的复杂文本整形——各模块的确切支持矩阵见其对应页面。

## 安装 {#install}

```sh
go get github.com/go-opentype/opentype
go get github.com/go-opentype/bidi
go get github.com/go-opentype/shape
go get github.com/go-opentype/fonts
```

## 接下来读什么 {#where-to-go-next}

- [opentype]({{< relref "opentype.md" >}})：sfnt 解析器与光栅化器，包括字体、`cmap`、TrueType `glyf` 与 CFF/CFF2 轮廓、可变字体、GSUB/GPOS 整形、字体提示（hinting）、垂直度量、用于数学排版的 `MATH` 表、字体子集化与描述符访问器，以及完整的支持矩阵。
- [bidi]({{< relref "bidi.md" >}})：UAX #9 引擎，以及它自己实现了什么、把什么交给整形器处理。
- [shape]({{< relref "shape.md" >}})：复杂文本整形器，包括阿拉伯文连写、印度系文字重排、通用整形引擎、埃及圣书体 quadrat、谚文、竖排、连字（ligature）、标记附着、字距调整（kerning），以及它覆盖哪些文字。
- [fonts]({{< relref "fonts.md" >}})：46 个内置字体族，以及用于获取它们的按字体族导入模型。
- **指南**：面向任务的分步讲解，包括将文本渲染为 PNG、整形混合方向的阿拉伯文字符串，以及挑选内置字体以获得良好的易读性。

源代码位于 [github.com/go-opentype](https://github.com/go-opentype)。
