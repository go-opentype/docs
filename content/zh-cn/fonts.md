---
title: "fonts"
weight: 50
---

# fonts：内置的易读字体族 {#fonts-bundled-legible-families}

`github.com/go-opentype/fonts` 提供 46 款易读、许可宽松的 TrueType 字体（font），每个字体族对应一个 Go 子包，各自带有独立的 `//go:embed`：无需下载，也无需自己寻找 `.ttf`，并且**你的二进制文件只会链接你所导入的字体族**。覆盖范围包括拉丁文、西里尔文和希腊文，外加七种非拉丁文字：阿拉伯文和希伯来文（RTL）、天城文、泰文、格鲁吉亚文、亚美尼亚文和埃及圣书体；另有用于 CJK（汉字、假名、谚文和常用 CJK 标点）的 Noto Sans SC、JP 和 KR。它是为纯 Go、仅依赖标准库的 TrueType 引擎 [opentype]({{< relref "opentype.md" >}}) 而构建的，但这些原始字节可用于任何接受 `.ttf` 的解析器。

```go
import (
	"github.com/go-opentype/fonts"
	"github.com/go-opentype/fonts/inter"
)

f, err := fonts.Parse(inter.TTF) // *opentype.Font
face := f.NewFace(16)            // 16px face
```

纯 Go，无 cgo，构建时和运行时都无需获取任何外部资源：每个字体族都通过 `//go:embed` 嵌入自己的子包。

## 编译期惰性的导入模型 {#the-lazy-at-compile-time-import-model}

`//go:embed` 是*按包*急切（eager）嵌入的：任何嵌入了字体的包，都会把该字体的字节链接进每个导入它的二进制文件，不论该二进制文件是否真的用到它。如果 `fonts` 包把全部 46 个字体族批量嵌入，那么你只要为任何目的导入它，这 46 个就会全部进入你的二进制文件。

因此根 `fonts` 包并不这样做。每个字体族位于各自的子包中，例如 `fonts/inter`、`fonts/roboto`、`fonts/jetbrainsmono` 等，并带有自己的 `//go:embed`。导入 `fonts/inter` 只会链接 Inter。导入十个子包，就只链接这十个。

根 `fonts` 包只包含两样东西：

1. **[`Family`](#api) 元数据**：每个内置字体的名称、[`Kind`](#api)、许可证和导入路径，由 `All` 和 `ByName` 返回。没有 `[]byte` 字段：枚举字体族绝不会把其中任何一个链接进来。
2. **[`MostLegible`](#api)**：直接嵌入根包的唯一一个字体族，因此无需额外导入就有一个合理的默认值。其他所有字体族都需要导入各自的子包。

## 内置字体 {#bundled-fonts}

六个字体族是手工挑选的（自 v0.1.0 起就有）；其余四十个是从 [google/fonts](https://github.com/google/fonts) 摄取的。七个非拉丁字体族（阿拉伯文、希伯来文、天城文、泰文、格鲁吉亚文、亚美尼亚文、埃及圣书体）以及 Noto Sans SC、JP 和 KR（内置的 CJK 字体族），使文字覆盖范围在拉丁文/西里尔文/希腊文之外更加完整。

| 名称 | 类别 | 许可证 | 导入路径 |
| --- | --- | --- | --- |
| Atkinson Hyperlegible | Sans | OFL-1.1 | `github.com/go-opentype/fonts/atkinsonhyperlegible` |
| Inter | Sans | OFL-1.1 | `github.com/go-opentype/fonts/inter` |
| Go | Sans | BSD-3-Clause | `github.com/go-opentype/fonts/goregular` |
| Lora | Serif | OFL-1.1 | `github.com/go-opentype/fonts/lora` |
| Go Mono | Mono | BSD-3-Clause | `github.com/go-opentype/fonts/gomono` |
| JetBrains Mono | Mono | OFL-1.1 | `github.com/go-opentype/fonts/jetbrainsmono` |
| Noto Sans Arabic (RTL) | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansarabic` |
| Noto Sans Hebrew (RTL) | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosanshebrew` |
| Noto Sans Devanagari | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansdevanagari` |
| Noto Sans Thai | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansthai` |
| Noto Sans Georgian | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansgeorgian` |
| Noto Sans Armenian | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansarmenian` |
| Noto Sans Egyptian Hieroglyphs | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansegyptianhieroglyphs` |
| Noto Sans SC (CJK) | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosanssc` |
| Noto Sans JP (CJK) | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansjp` |
| Noto Sans KR (CJK) | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosanskr` |
| … 另有 30 个拉丁文/西里尔文/希腊文字体族（Roboto、Open Sans、Montserrat、Fira Sans/Code、IBM Plex 等） | Sans/Serif/Mono | OFL-1.1 | 完整列表见 [pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts) |

完整的许可证文本原样收录在 `licenses/` 下，每个字体族一个文件（每个 `OFL-1.1` 行对应 SIL 开放字体许可证 1.1，另有 Go 和 Go Mono 共用的 `GoFonts-LICENSE.txt`，即 Bigelow & Holmes 的 BSD-3-Clause）。

{{< callout type="info" >}}
**默认字体：Atkinson Hyperlegible**

由盲文研究所（Braille Institute）专为最大限度区分字符而设计，面向低视力读者，同时也让所有读者受益。[`MostLegible()`](#api) 返回的就是它，它也是唯一直接嵌入根包的字体族。
{{< /callout >}}

{{< callout type="info" >}}
**内置的实例是静态的；引擎本身支持可变字体**

**Inter**、**Lora**、**JetBrains Mono** 以及大部分从 `google/fonts` 摄取的字体，在上游都是可变字体；每个内置的 `.ttf` 都固定在该字体族的默认主字形（静态实例）上，以保持打包简单、体积小。这是打包上的选择，并非引擎的限制：[opentype]({{< relref "opentype.md#variable-fonts" >}}) 通过 `(*Face).SetVariation` 完整支持 OpenType 可变字体。如果你需要特定的轴坐标（例如字重 600 的 Inter），请自行获取该字体族的可变 `.ttf`，并用 `opentype.Parse` 解析。
{{< /callout >}}

## API {#api}

```go
type Kind int

const (
	KindSans Kind = iota
	KindSerif
	KindMono
	KindDisplay
)

// Family is metadata only — no font bytes. Fetch bytes via ImportPath.
type Family struct {
	Name       string // e.g. "Inter"
	Kind       Kind
	License    string // SPDX id: "OFL-1.1" or "BSD-3-Clause"
	ImportPath string // e.g. "github.com/go-opentype/fonts/inter"
}

func All() []Family                        // every bundled family's metadata, stable order
func ByName(name string) (Family, bool)     // case-insensitive lookup by Family.Name
func MostLegible() []byte                   // Atkinson Hyperlegible bytes — the one family embedded here
func Parse(ttf []byte) (*opentype.Font, error) // convenience wrapper over opentype.Parse
```

其他每个字体族的字节都放在各自的子包中，形式为 `var TTF []byte`，例如 `inter.TTF`、`roboto.TTF`、`jetbrainsmono.TTF`，完整的导入路径列表见上表和 [pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts)。

## 许可证 {#license}

仓库中的 Go 源代码（内置 `.ttf` 文件和 `licenses/` 之外的所有内容）采用 BSD-3-Clause。内置的字体文件保留各自的上游许可证；完整文本、版权声明以及（OFL 字体的）保留字体名称（Reserved Font Names），请参见上表和 `licenses/`。

源代码：[github.com/go-opentype/fonts](https://github.com/go-opentype/fonts) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts)
