---
title: "opentype"
weight: 20
---

# opentype：引擎 {#opentype-the-engine}

`github.com/go-opentype/opentype` 是 TrueType/OpenType 字体的纯 Go 解析器和抗锯齿光栅化器（rasteriser），使用 `CGO_ENABLED=0` 构建，**仅依赖标准库**。它只导入 `image`、`encoding/binary`、`math`、`errors` 等包，没有 `golang.org/x/*`，也没有第三方模块，因此可为所有 Go 目标平台构建，包括 `GOOS=js GOARCH=wasm`。

它的目的是在功能上取代 `golang.org/x/image/font/opentype` 中字形绘制型界面所需的那一小部分：解析字体数据、按像素大小构建字面（face）、获取每个 rune 的步进宽度和 8 位 alpha 覆盖蒙版，并且在此基础上走得更远：CFF/CFF2 轮廓、可变字体、GPOS/GSUB 整形、字距调整（kerning）以及 TrueType/CFF 字体提示（hinting）都在其范围之内。

## 用法 {#usage}

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

`GlyphMask` 的形式有意仿照 `x/image` 的 `font.Face.Glyph`，因此用它替换 x/image 的 face 是机械式的。

## 可变字体 {#variable-fonts}

用 `SetVariation`，按轴标签（`wght`、`wdth`、`opsz` 等）为键，把可变字体实例化到其设计空间中的某个具体位置：

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

`SetVariation` 会重新插值字形（glyph）轮廓（TrueType 用 `gvar`，CFF2 用 CFF2 混合），并针对所选坐标重新推导由 `HVAR`/`VVAR`/`MVAR` 驱动的度量；`avar` 的轴值映射会自动应用。

## 字体提示 {#hinting}

`SetHinting(true)` 会在光栅化之前运行 TrueType 指令解释器（用于 `glyf` 轮廓）或 CFF/Type 2 笔画网格适配器（用于 CFF/CFF2 轮廓）；`SetStemDarkening(true)` 还会额外应用 CFF 笔画加深。两者默认均为关闭（统一的 4×4 超采样渲染）。

## 数学排版（`MATH` 表）{#mathematical-typesetting-the-math-table}

`opentype` 解码 OpenType 的 **`MATH`** 表：这是数学排版引擎（LuaTeX、unicode-math、MathJax、Word 的公式编辑器，或任何基于 HarfBuzz 的 `hb-ot-math` 构建的软件）排布公式所需的字体级度量。它由三部分组成，全部通过 `Face` 以像素缩放后的形式暴露：

- **MathConstants**：约 56 个全局标量：数学轴高度、下标/上标的位移与间距下限、分数线和根号的间隙、线条粗细等等，通过 `MathConstant` 以及 `opentype.AxisHeight` 或 `opentype.FractionRuleThickness` 之类的 `MathConstant` 值来读取（完整的、按磁盘顺序排列的列表请参见 `go doc`）。
- **MathGlyphInfo**：每个字形的*斜体修正*（`ItalicCorrection`，即在倾斜字形之后、直立字形之前应插入的空间，或跟在上标之后的下标的水平偏移）、*顶部重音附着点*（`TopAccentAttachment`，即把重音符号居中放在字形之上的 x 位置）、*扩展形状*标志（`IsExtendedShapeGlyph`，用于例如大型积分号这类字形，其上标的定位如同该字形被拉伸过一样），以及四角切入式*数学字距调整*（`MathKern`，是相邻上下标附着高度的阶梯函数）。
- **MathVariants**：字形的更大尺寸变体；当最大的变体仍不够大时，则提供用可重复的部件组装出任意高或任意宽的可伸缩字形（积分号、大括号、根号、箭头）的配方，两者由 `MathVariants` 一并返回。

数学*布局*本身，即盒模型、TeX 的数学列表规则、选择变体或把组装件增长到目标尺寸，是建立在这些度量之上的更高层问题，不属于本包的范围，正如 `hb-ot-math` 只暴露度量，而把布局留给调用方。

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

在真实的 STIX Two Math 字体上以每 em 32 像素运行时，它会输出 `axis
height: 8 fraction rule thickness: 2`、`italic correction: 1` 以及 `size
variants: 13 assembly parts: 3`。STIX Two Math（SIL 开放字体许可证）作为 [`testdata/STIXTwoMath-Regular.otf`](https://github.com/go-opentype/opentype/blob/main/testdata/STIXTwoMath-Regular.otf) 附带在 `opentype` 仓库中，并在 CI 中进行端到端检查，同时还有合成的 MATH 表固件，使解码器达到 100% 的分支覆盖率。

## 字体描述符与子集化 {#font-descriptors-and-subsetting}

对于需要*嵌入*字体的程序，例如 PDF 写入器、CSS `@font-face` 生成器，`opentype` 也暴露了写入端所需的原语：

- 来自 OS/2 和 `head`/`post` 的**描述符标量**，可直接用于填写 PDF 的 `/FontDescriptor` 或 CSS 声明：`Ascent`、`Descent`、`LineGap`、`FontBBox`、`CapHeight`、`XHeight`、`ItalicAngle`、`WeightClass`、`WidthClass`、`StemV`、`Flags`，以及 `IsFixedPitch`/`IsItalic`/`IsSerif` 这几个布尔值。
- **实例化**：`Instance(coords)` 返回一个新的 `*Font`，固化在可变字体设计空间中的单个点上（glyf/CFF2 轮廓和度量被冻结），`InstanceBytes(coords)` 则把该实例作为全新的 sfnt 数据块返回。
- **子集化**：`SubsetTrueType(gids)` 把 TrueType 字体裁剪到所请求的字形，闭合复合字形的依赖，并返回旧到新的字形 ID 重映射；`SubsetCFF(gids)` 对 CFF/OTTO 字体做同样的事，并闭合 `seac` 重音引用。两者都只保留文档实际用到的字形，使嵌入的字体保持小巧。

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

## WOFF2 与二进制体积 {#woff2-and-binary-size}

**状态：已在 main 上，将随下一次发布**（已合并，尚未打标签）。

WOFF2 字体现在使用只含解码器的 `github.com/dsnet/compress/brotli` 来解压，不再使用 `github.com/andybalholm/brotli`。`andybalholm/brotli` 的根包导入了 `net/http`，因此 `net/http` 和 `crypto/tls` 的 `init` 被链接进了每个导入 `opentype` 的程序：在一个 7.5 MB 的 `GOOS=js` 文本程序中占了 1.8 MB。只含解码器的包两者都不导入。

现在，解压受 WOFF2 目录所声明的大小限制，最多 256 MiB，因此解压炸弹会被拒绝，而不是被分配内存。

未决事项（这是一次权衡，而不是修复）：`dsnet/compress/brotli` 的解压速度慢约 1.37 倍，对于 200 KB 的字体大约是 0.4 毫秒。为了体积和上限而接受了这一点，并记录在此，以便日后重新评估。

## API {#api}

| 符号 | 用途 |
| --- | --- |
| `Parse(b []byte) (*Font, error)` | 解码 sfnt（TrueType 或 CFF/OTTO）字体数据块。 |
| `(*Font).NumGlyphs() int` | 字形数量（来自 `maxp`）。 |
| `(*Font).GlyphIndex(r rune) (GlyphIndex, bool)` | 通过 cmap 映射一个 rune。 |
| `(*Font).GlyphIndexVariation(r, vs rune) (GlyphIndex, bool)` | 映射一个 rune 加 Unicode 变体选择符。 |
| `(*Font).Axes() []Axis` | 可变字体的设计轴（`fvar`）。 |
| `(*Font).NamedInstances() []NamedInstance` | 变体空间中的命名位置。 |
| `(*Font).GPOS() *GPOS` / `(*Font).GSUB() *GSUB` | 已解析的布局表，或 `nil`。 |
| `(*Font).HasVerticalMetrics() bool` | 是否存在 `vhea`/`vmtx`。 |
| `(*Font).NewFace(sizePx int) *Face` | 构建指定大小的 face（缩放 = `sizePx / unitsPerEm`）。 |
| `(*Face).Metrics() Metrics` | 以像素为单位的 `Ascent`、`Descent`、`Height`。 |
| `(*Face).Advance(r rune) int` / `(*Face).Measure(s string) int` | rune / 字符串的步进宽度（像素）。 |
| `(*Face).Kern(prev, r rune) int` / `(*Face).MeasureKerned(s string) int` | GPOS 或 `kern` 的字对调整。 |
| `(*Face).GlyphMask(r, x, y) (image.Rectangle, *image.Alpha, image.Point, int, bool)` | 光栅化后的字形与步进宽度。 |
| `(*Face).Shape(text string, features ...string) []GlyphIndex` | 经 GSUB 替换后的字形序列。 |
| `(*Face).ShapePositioned(text string, features ...string) []PositionedGlyph` | 经 GSUB + GPOS 处理、带笔位偏移的字形序列。 |
| `(*Face).SetVariation(coords map[string]float64)` | 在给定的轴坐标处实例化可变字体。 |
| `(*Face).SetHinting(on bool)` / `(*Face).SetStemDarkening(on bool)` | 开关 TrueType/CFF 字体提示器。 |
| `(*Face).VerticalAdvance(r rune) int` / `(*Face).VerticalOrigin(r rune) (int, bool)` | 竖排书写模式的度量。 |
| `(*Font).HasMath() bool` | 字体是否带有 OpenType `MATH` 表。 |
| `(*Font).IsExtendedShapeGlyph(gid GlyphIndex) bool` | gid 是否在 `MATH` 扩展形状覆盖范围内。 |
| `(*Face).MathConstant(which MathConstant) int` | 约 56 个按像素缩放的 `MATH` 布局常量之一（轴高度、上下标位移、分数线/根号间隙等）。 |
| `(*Face).ItalicCorrection(gid GlyphIndex) int` | 每个字形的数学斜体修正，单位为像素。 |
| `(*Face).TopAccentAttachment(gid GlyphIndex) (int, bool)` | 把顶部重音符号居中放在 gid 之上的 X 位置（像素）。 |
| `(*Face).MathKern(gid GlyphIndex, corner MathKernCorner, correctionHeight int) int` | gid 四个角之一在给定附着高度处的切入式数学字距调整（像素）。 |
| `(*Face).MathVariants(gid GlyphIndex, vertical bool) ([]MathVariant, *MathAssembly)` | gid 沿某一轴的尺寸变体及可伸缩字形的组装件。 |
| `(*Font).Ascent/Descent/LineGap/FontBBox/CapHeight/XHeight/ItalicAngle/WeightClass/WidthClass/StemV/Flags/IsFixedPitch/IsItalic/IsSerif` | 来自 OS/2 + `head`/`post` 的字体描述符标量，用于 PDF `/FontDescriptor` 和 CSS `@font-face`。 |
| `(*Font).Instance(coords map[string]float64) (*Font, error)` / `(*Font).InstanceBytes(coords) ([]byte, error)` | 将可变字体固化为静态实例（`*Font` 或新的 sfnt 数据块）。 |
| `(*Font).SubsetTrueType(gids []GlyphIndex) ([]byte, map[GlyphIndex]GlyphIndex, error)` | 将 TrueType 字体子集化到一组字形，闭合复合字形；返回旧到新的 ID 映射。 |
| `(*Font).SubsetCFF(gids []GlyphIndex) ([]byte, error)` | 将 CFF/OTTO 字体子集化到一组字形，闭合 `seac` 重音。 |

`Font` 在 `Parse` 之后是不可变的，可安全地并发使用。`Face` 会缓存已光栅化的字形，**不**可安全地并发使用；如有需要，请为每个 goroutine 各构建一个 `Face`。

## 支持矩阵 {#support-matrix}

- sfnt 容器（`0x00010000` 和 `true` 两种 TrueType 魔数，以及 `OTTO` CFF 魔数）
- `head`、`maxp`、`hhea`、`hmtx`（包括末尾连续字形共用步进宽度的情形）
- `cmap` 格式 **0、2、4、6、8、10、12、13** 和 **14**（Unicode 变体序列），优先使用 12
- `loca`（短格式和长格式）、`glyf` 简单字形（重复标志、短/长增量）和复合字形（`ARGS_ARE_XY_VALUES`、缩放 / x&y 缩放 / 2×2、`MORE_COMPONENTS`，带有环路和深度保护）
- 为二次轮廓合成隐含的曲线上中点
- CFF 和 CFF2（`OTTO`）Type 2 charstring 轮廓，包括子程序、`seac` 重音组合，以及 CFF2 可变字体的混合运算符
- `fvar` 轴与命名实例、`avar` 轴值映射、`gvar`/CFF2 字形轮廓插值、`HVAR`/`VVAR`/`MVAR` 度量变化，均通过 `(*Face).SetVariation`
- 所有 `GSUB`/`GPOS` 查找类型：连字（ligature）、上下文替换和位置形式替换（`isol`/`init`/`medi`/`fina`）、单字形/字对/连写定位、mark-to-base 和 mark-to-mark 附着，外加 `GDEF` 和查找标志的标记过滤
- 通过 GPOS 字对定位进行字距调整，并以旧式 `kern` 表作为回退，统一经由一个 `Kerner`
- TrueType 指令字体提示器和 CFF/Type 2 笔画网格适配器（蓝区、字腔、笔画加深），按 `Face` 用 `SetHinting` / `SetStemDarkening` 切换
- 感知 `vhea`/`vmtx`/`VORG` 的垂直度量和原点，用于竖排书写模式
- OpenType **`MATH`** 表：数学排版度量（常量、每个字形的斜体修正 / 顶部重音附着 / 四角字距调整，以及可伸缩字形的尺寸变体和组装件），通过 `Face` 以像素缩放形式暴露（`HasMath`、`MathConstant`、`ItalicCorrection`、`TopAccentAttachment`、`MathKern`、`MathVariants`、`IsExtendedShapeGlyph`）；数学*布局*留给更高层的引擎
- 来自 OS/2 + `head`/`post` 的**字体描述符**标量（`Ascent`、`Descent`、`FontBBox`、`CapHeight`、`XHeight`、`ItalicAngle`、`WeightClass`、`WidthClass`、`StemV`、`Flags` 等），用于 PDF 和 CSS 嵌入
- 将可变字体**实例化**为静态主字形（`Instance`、`InstanceBytes`），以及**子集化**到一组字形（`SubsetTrueType` 带复合字形闭合和旧到新的 ID 映射，`SubsetCFF` 带 `seac` 闭合）
- 在非零环绕规则下，通过 4×4 超采样实现抗锯齿光栅化

光栅化采用统一的超采样（而不是原生 TrueType/CFF 字体提示器那种基于增量提示的丢点控制光栅化），因此它优先保证正确性和可移植性，而不去追求小字号下最后一点锐利度。在 HarfBuzz + FreeType 所覆盖的路径上，这是一个功能完整的引擎：上面列出的内容没有一项是桩实现。

## 测试 {#testing}

测试从不依赖外部字体来达到 100% 覆盖率：它们在内存中合成最小但有效的 TrueType 和 CFF 字体（表目录 + `head`/`maxp`/`hhea`/`hmtx`/`cmap`/`loca`/`glyf`，或合成的 `MATH` 表），以确定性地覆盖每一个解析和光栅化分支，包括错误路径。真实字体，即 Adobe 的 Source Serif 4，以及（针对 `MATH` 表的）STIX Two Math（两者均为 SIL 开放字体许可证，附带在 `testdata/` 下），另外在 `example_test.go` 和少数健全性检查测试中做端到端验证，使文档中的示例同时充当针对生产字体数据的冒烟测试。CI 强制要求**100.0% 的语句覆盖率**、`go vet`，以及对 `linux/{amd64,arm64,riscv64,loong64,ppc64le,s390x}`、`js/wasm`、`darwin/arm64` 和 `windows/amd64` 的交叉编译冒烟测试。

源代码：[github.com/go-opentype/opentype](https://github.com/go-opentype/opentype) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/opentype)
