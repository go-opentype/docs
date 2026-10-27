---
title: "複雑なスクリプトをシェーピングする"
weight: 80
---

# ガイド: 複雑なスクリプトをシェーピングする {#guide-shape-complex-scripts}

ルーンごとにアドバンスを足していく単純なループ（[テキストを PNG に描画する]({{< relref "render-text-to-png.md" >}}) のもの）は、孤立したラテン文字のテキストには十分ですが、**並べ替え**や**連綿結合**が必要なテキストでは破綻します。たとえば、右から左へ書かれるアラビア語の連なりや、隣の文字とつながるべきアラビア文字がそうです。そこで使うのが [`shape`]({{< relref "../shape.md" >}}) です。

## 右から左へ書くアラビア語の文字列 {#a-right-to-left-arabic-string}

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

元の文字列は論理順では右から左ですが、`shape.Shape` はグリフをすでに**視覚順**、つまり描画順で左から右に並べて返します。この用途で [`bidi`]({{< relref "../bidi.md" >}}) を自分で呼び出す必要はありません。`shape` が内部で組み合わせています。

## 方向が混在するテキスト {#mixed-direction-text}

複数のスクリプトが混在する段落（アラビア語の中に英語が埋め込まれている場合や、その逆の場合）では、`Options.Direction` をデフォルトの `Auto` のままにしてください。基本方向は UAX #9 の規則 P2/P3 に従って最初の強方向文字から選ばれ、文字列内の各方向のランはそれぞれ独立にシェーピングと並べ替えが行われた後、視覚順に連結されます。

```go
glyphs := shape.Shape(face, "The word بيت means house", shape.Options{})
```

段落（または埋め込み）の方向を自動検出に任せず固定したい場合、たとえば先頭の文字に関係なく常に RTL でレイアウトする UI では、明示的に設定します。

```go
glyphs := shape.Shape(face, text, shape.Options{
    Direction: bidi.RightToLeft,
})
```

## スクリプトを強制する {#forcing-a-script}

スクリプトは自動検出されます（アラビア文字ブロックのルーンが 1 つでもあればアラビア語のシェーパが選ばれ、それ以外は Latin/default の経路に入ります）。自動検出よりも自分のほうが事情を把握している場合は、`Options.Script` で上書きしてください。たとえば、ラテン文字だけの UI 文字列にアラビア語の句読点がたまたま含まれていて、それによる連綿結合を起こしたくない場合です。

```go
glyphs := shape.Shape(face, text, shape.Options{Script: "latn"})
```

## インド系文字と Universal Shaping Engine のスクリプト {#indic-and-universal-shaping-engine-scripts}

デーヴァナーガリー、ベンガル文字、タミル文字など、専用のシェーパを持つ 10 のインド系スクリプトでは、HarfBuzz の「indic」モデルがフルに適用されます。音節の分割、ベースとレフの検出、前置マートラとレフの並べ替え、そして（mark/mkmk/abvm/blwm による配置を含む）階層化された GSUB/GPOS フィーチャーパイプラインです。

```go
glyphs := shape.Shape(face, "नमस्ते", shape.Options{}) // Devanagari "hello"
```

タイ語、ラオ語、クメール語、ミャンマー語、チベット語など、専用のシェーパを持たないスクリプトは Universal Shaping Engine（USE）で処理されます。USE は、連なりを音節カテゴリに分類してクラスタに分割し、前置母音や修飾記号とレファを並べ替え、USE の GSUB/GPOS パイプライン（サコット/ハラントの結合、分割母音の分解、不完全なクラスタへのドット付き円の挿入を含む）を実行します。スクリプトの検出は自動なので、`shape.Shape` を呼び出す以外にコードを変える必要はありません。それぞれのモデルが対象とするスクリプトの全一覧は、[shape の対応範囲]({{< relref "../shape.md#scope" >}})を参照してください。

## 縦書き（CJK の tategaki）とその他のスクリプト {#vertical-cjk-tategaki-and-other-scripts}

エジプト聖刻文字のクワドラットとハングルのジャモ合成も、入力テキストのスクリプトから自動的に処理されます。縦書きモード（CJK の tategaki、上から下へ、`vert`/`vrt2` のグリフ形）にするには、`Options.Vertical` を設定します。

```go
glyphs := shape.Shape(face, "こんにちは", shape.Options{Vertical: true})
```

`shape.Shape` は、これらすべての経路（アラビア語、インド系、USE、エジプト聖刻文字、ハングル、縦書き、そして Latin/default（キリル文字、ギリシャ文字、横書きの CJK も含みます））にわたって完結したソリューションです。
