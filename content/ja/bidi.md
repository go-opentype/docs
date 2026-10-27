---
title: "bidi"
weight: 30
---

# bidi — Unicode 双方向アルゴリズム {#bidi-the-unicode-bidirectional-algorithm}

`github.com/go-typeset/bidi` は、純粋な Go 製、`CGO_ENABLED=0`、**標準ライブラリのみ**で実装された
[Unicode 双方向アルゴリズム（UAX #9）](https://www.unicode.org/reports/tr9/)です。左から右と右から左が混在するテキストのレイアウトに使います。

Go 製の多くの双方向テキスト実装とは異なり、`golang.org/x/text` には**依存しません**。`Bidi_Class` と対になる括弧の Unicode プロパティは、生成された小さなルックアップテーブルにコンパイルされています（[`cmd/genbidi`](https://github.com/go-typeset/bidi/tree/main/cmd/genbidi) を参照）。そのため、標準ライブラリが動く環境ならどこでもビルドできます。

{{< callout type="warning" >}}
**移動しました。古いインポートパスは失敗します**


このパッケージは以前 `github.com/go-opentype/bidi` でした。移管の際にリダイレクトが残されているため、古い**リンク**は今でも開けますが、古い**インポート**は動作しません。モジュールが宣言しているパスは新しいものだからです。

```
$ go get github.com/go-opentype/bidi@latest
module declares its path as: github.com/go-typeset/bidi
        but was required as: github.com/go-opentype/bidi
```
{{< /callout >}}

## 使い方 {#usage}

```go
package main

import (
	"fmt"

	"github.com/go-typeset/bidi"
)

func main() {
	// A logical-order string mixing English and Hebrew.
	s := "abc אבג def"

	// Visual (left-to-right) order for display.
	fmt.Println(bidi.VisualOrder(s, bidi.Auto))

	// Or work with levels directly.
	runes := []rune(s)
	levels := bidi.ResolveLevels(runes, bidi.LeftToRight)
	order := bidi.Reorder(runes, levels) // visual-order permutation of indices
	fmt.Println(levels, order)

	// Inspect a single rune's Bidi_Class.
	fmt.Println(bidi.ClassOf('א')) // R
}
```

## API {#api}

| Symbol | Purpose |
| --- | --- |
| `ClassOf(r rune) Class` | ルーンの Bidi_Class |
| `Class` enum | `L R AL EN ES ET AN CS NSM BN B S WS ON LRE RLE LRO RLO PDF LRI RLI FSI PDI` |
| `ResolveLevels(text []rune, base Direction) []Level` | ルーンごとに解決された埋め込みレベル |
| `BaseLevel(text []rune, base Direction) Level` | 段落レベル（規則 P2/P3） |
| `Reorder(text []rune, levels []Level) []int` | 規則 L2 による視覚順の並べ替え（順列） |
| `VisualOrder(text string, base Direction) string` | 解決と並べ替えをまとめて行う便利関数 |
| `Direction` | `LeftToRight`、`RightToLeft`、`Auto` |
| `Level` | 埋め込みレベル（偶数 = LTR、奇数 = RTL） |

## 実装済みの範囲と先送りした範囲 {#implemented-vs-deferred}

**実装済み** — アルゴリズムは規則 **L2** まで動作します。これは Unicode の適合性テストファイル `BidiCharacterTest.txt` がカバーする範囲のすべてです。

- **P2、P3** 最初の強方向文字から決まる基本の段落レベル。
- **X1–X8** 明示的な埋め込み*および*アイソレート（オーバーフローの処理と方向ステータススタックを含む）。**X9** 非推奨の書式制御文字と `BN` の除去。**X10** `sos`/`eos` を伴うアイソレーティングランシーケンス。
- **W1–W7** 弱い型。
- **N0** 対になる括弧の解決（BD16。U+2329/U+232A の正規等価を含む）、**N1–N2** 中立型。
- **I1、I2** 暗黙のレベル。
- **L1** 区切り文字と末尾の空白のリセット、**L2** 並べ替え。

**先送りした範囲**（双方向エンジンの責務の外にあり、シェーパの仕事です）:

- **L3**（結合マーク）と **L4**（対になる括弧やその他の鏡像文字のグリフ鏡像化）— `VisualOrder` は鏡像グリフへの置換を行いません。これは描画・シェーピングの段階に属します。
- アラビア文字の連綿**シェーピング／結合** — [shape]({{< relref "shape.md" >}}) を参照してください。
- **P1** 段落の分割 — 呼び出し側が行います。API は段落単位で動作します（段落内にある `Paragraph_Separator` は、引き続き X8/L1 で処理されます）。

## 適合性 {#conformance}

このパッケージは `BidiCharacterTest.txt` の**全体**に対して検証されています（段落レベル、文字ごとのレベル、視覚順のすべてのケースが合格します）。代表的なケースを選んだサブセットが `testdata` 以下に埋め込まれており、`TestConformance` で実行されます。CI では、ステートメントカバレッジが**ちょうど 100%** であること、`go vet`、`gofmt`、そして 6 つの 64 ビットアーキテクチャと `js/wasm`、`darwin/arm64`、`windows/amd64` へのクロスコンパイルを必須としています。

## テーブルの再生成 {#regenerating-the-tables}

```sh
go run ./cmd/genbidi .
```

このコマンドは、Unicode Character Database から最新の `DerivedBidiClass.txt` と `BidiBrackets.txt` を取得し、`bidiclass_table.go` と `bidibrackets_table.go` を書き換えます。

ソース: [github.com/go-typeset/bidi](https://github.com/go-typeset/bidi) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-typeset/bidi)
