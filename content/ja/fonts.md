---
title: "fonts"
weight: 50
---

# fonts — 読みやすさを重視した同梱ファミリー {#fonts-bundled-legible-families}

`github.com/go-opentype/fonts` は、読みやすく、寛容なライセンスの TrueType フォントを 46 種類提供します。ファミリーごとに Go のサブパッケージを 1 つ用意し、それぞれが独自の `//go:embed` を持ちます。ダウンロードも `.ttf` の調達も不要で、**バイナリにリンクされるのは、インポートしたファミリーだけ**です。対応範囲は、ラテン文字、キリル文字、ギリシャ文字に加えて 7 つの非ラテン文字、すなわちアラビア文字とヘブライ文字（RTL）、デーヴァナーガリー、タイ文字、ジョージア文字、アルメニア文字、エジプト聖刻文字、さらに CJK 向けの Noto Sans SC、JP、KR（漢字、かな、ハングル、よく使う CJK の約物）です。純粋な Go 製で標準ライブラリのみを使う TrueType エンジンである [opentype]({{< relref "opentype.md" >}}) 向けに作られていますが、生のバイト列は `.ttf` を受け付ける任意のパーサで使えます。

```go
import (
	"github.com/go-opentype/fonts"
	"github.com/go-opentype/fonts/inter"
)

f, err := fonts.Parse(inter.TTF) // *opentype.Font
face := f.NewFace(16)            // 16px face
```

純粋な Go 製で cgo は不要です。ビルド時にも実行時にも取得すべき外部アセットはなく、すべてのファミリーが専用のサブパッケージに `//go:embed` で埋め込まれています。

## コンパイル時に遅延させるインポートモデル {#the-lazy-at-compile-time-import-model}

`//go:embed` は*パッケージ単位*で先行的に働きます。フォントを埋め込んだパッケージは、使うかどうかに関わらず、そのパッケージをインポートするすべてのバイナリにフォントのバイト列をリンクします。46 のファミリーすべてを一括で埋め込む `fonts` パッケージだとすると、何かのためにインポートしただけで、46 すべてがバイナリに入ってしまいます。

そこで、ルートの `fonts` パッケージはそうしていません。各ファミリーは、`fonts/inter`、`fonts/roboto`、`fonts/jetbrainsmono` のように専用のサブパッケージに置かれ、それぞれが独自の `//go:embed` を持ちます。`fonts/inter` をインポートしてリンクされるのは Inter だけで、10 個のサブパッケージをインポートすれば、その 10 個だけがリンクされます。

ルートの `fonts` パッケージが持つのは、次の 2 つだけです。

1. **[`Family`](#api) メタデータ** — 同梱されている各フォントの名前、[`Kind`](#api)、ライセンス、インポートパスです。`All` と `ByName` が返します。`[]byte` フィールドはないため、ファミリーを列挙してもバイト列がリンクされることはありません。
2. **[`MostLegible`](#api)** — ルートパッケージに直接埋め込まれている唯一のファミリーで、追加のインポートなしで使える妥当なデフォルトです。それ以外のファミリーは、それぞれのサブパッケージをインポートする必要があります。

## 同梱フォント {#bundled-fonts}

6 つのファミリーは手作業で選定したもので（v0.1.0 から存在します）、残りの 40 は [google/fonts](https://github.com/google/fonts) から取り込んだものです。7 つの非ラテン文字ファミリー（アラビア文字、ヘブライ文字、デーヴァナーガリー、タイ文字、ジョージア文字、アルメニア文字、エジプト聖刻文字）と Noto Sans SC、JP、KR（同梱の CJK ファミリー）が、ラテン文字・キリル文字・ギリシャ文字以外のスクリプトへの対応を補っています。

| Name | Kind | License | Import path |
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
| … ほかに 30 のラテン文字・キリル文字・ギリシャ文字ファミリー（Roboto、Open Sans、Montserrat、Fira Sans/Code、IBM Plex など） | Sans/Serif/Mono | OFL-1.1 | 全一覧は [pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts) を参照してください |

ライセンス全文は `licenses/` 以下にそのまま同梱されており、ファミリーごとに 1 ファイルあります（`OFL-1.1` の行にはすべて SIL Open Font License 1.1、それに加えて Go と Go Mono が共有する `GoFonts-LICENSE.txt`（BSD-3-Clause、Bigelow & Holmes）があります）。

{{< callout type="info" >}}
**デフォルト: Atkinson Hyperlegible**

Braille Institute が、弱視の読者にとって文字の区別がつきやすくなるよう特に設計したフォントで、結果としてすべての読者の役に立ちます。[`MostLegible()`](#api) はこのフォントを返し、ルートパッケージに直接埋め込まれている唯一のファミリーです。
{{< /callout >}}

{{< callout type="info" >}}
**同梱インスタンスは静的ですが、エンジン自体は可変フォントに対応しています**

**Inter**、**Lora**、**JetBrains Mono**、および `google/fonts` から取り込んだもののほとんどは、上流では可変フォントです。同梱の各 `.ttf` は、バンドルを単純かつ小さく保つために、そのファミリーのデフォルトマスター（静的インスタンス）に固定されています。これはパッケージング上の選択であり、エンジンの制約ではありません。[opentype]({{< relref "opentype.md#variable-fonts" >}}) は `(*Face).SetVariation` によって OpenType Variations に完全に対応しています。特定の軸の値（たとえば Inter のウェイト 600）が必要な場合は、そのファミリーの可変 `.ttf` を自分で入手し、`opentype.Parse` で解析してください。
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

それ以外のファミリーのバイト列は、それぞれのサブパッケージに `var TTF []byte` として置かれています。たとえば `inter.TTF`、`roboto.TTF`、`jetbrainsmono.TTF` です。インポートパスの全一覧は、上の表と [pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts) を参照してください。

## ライセンス {#license}

リポジトリ内の Go ソースコード（同梱の `.ttf` ファイルと `licenses/` 以外のすべて）は BSD-3-Clause です。同梱のフォントファイルは、それぞれ上流のライセンスのままです。全文、著作権表示、（OFL フォントの）予約フォント名（Reserved Font Names）については、上の表と `licenses/` を参照してください。

ソース: [github.com/go-opentype/fonts](https://github.com/go-opentype/fonts) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts)
