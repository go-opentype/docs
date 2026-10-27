---
title: "shape"
weight: 40
---

# shape — 複雑テキストシェーパ {#shape-the-complex-text-shaper}

`github.com/go-opentype/shape` は、純粋な Go 製、`CGO_ENABLED=0`、**標準ライブラリのみ**で実装された複雑テキストのシェーパ（shaper）で、go-opentype スタックにおける HarfBuzz-lite です。Unicode テキストの連なりを、**視覚順**に並んだ配置済みのグリフ（glyph）へ変換します。これにより、アラビア語、インド系文字、東南アジアの文字、CJK の縦書き、エジプト聖刻文字のテキストが、孤立して結合されず並べ替えもされていないグリフとしてではなく、正しく描画されます。

組み合わせるのは兄弟モジュール 2 つだけで、ほかには何も加えません。[`bidi`]({{< relref "bidi.md" >}}) による双方向の並べ替えと、[`opentype`]({{< relref "opentype.md" >}}) による GSUB/GPOS に基づく置換と配置です。`golang.org/x/*` もサードパーティのモジュールも使わず、`GOOS=js GOARCH=wasm` を含むあらゆる Go ターゲットでビルドできます。

## できること {#what-it-does}

- **双方向の並べ替え** — UAX #9 の埋め込みレベルを（[`bidi`]({{< relref "bidi.md" >}}) 経由で）解決し、グリフを左から右へ並べます。そのため、右から左へ書かれる連なりは描画順に出力されます。
- **アラビア文字の連綿結合** — 各文字の結合形（孤立形・語頭形・語中形・語末形）を解決し、フォントの `isol`/`init`/`medi`/`fina` GSUB フィーチャーを、その形になっているグリフにだけ**位置ごとに**適用します。結合形は `ccmp` の分解を通じて追跡されます（Noto Sans Arabic のような実際のフォントが使う、ラスム骨格とドットの構成です）。そのため、結合されたグリフはきちんとつながります。
- **インド系文字のシェーピング** — デーヴァナーガリー、ベンガル文字、グルムキー、グジャラーティー、オリヤー、タミル文字、テルグ文字、カンナダ文字、マラヤーラム文字、シンハラ文字を、HarfBuzz の「indic」モデルで処理します。音節の分割、ベースとレフの検出、2 回の並べ替え（前置マートラとレフ）、基本フィーチャーと表示形フィーチャーを含む GSUB フィーチャーパイプライン全体、そして GPOS の mark/mkmk/abvm/blwm による配置が含まれます。
- **Universal Shaping Engine（USE）** — 専用のシェーパを持たないスクリプト向けの汎用の複雑スクリプトモデルです。タイ語、ラオ語、クメール語、ミャンマー語、チベット語、ジャワ語、バリ語、ブギス語、タイタム文字などが対象です。連なりを USE の音節カテゴリに分類してクラスタに分割し、並べ替え（前置母音と修飾記号をベースの前へ、レファをベースの後ろへ）を行った上で、USE の GSUB/GPOS パイプラインを通します。サコット/ハラントの結合、分割母音の分解、不完全なクラスタへのドット付き円の挿入も行います。
- **エジプト聖刻文字のクワドラット** — Unicode の書式制御文字 U+13430–U+1345F（結合子、角への挿入、重ね合わせ、セグメント/囲みの区切り）を二次元のクワドラット木として解析し、幾何学的にレイアウトします。そのため、文字の連なりがコンパクトなブロックとして描画されます。
- **ハングル** — ジャモを合成して、あらかじめ合成された音節ブロックにします。
- **縦書きモード（CJK の tategaki）** — `Options.Vertical` を指定すると、`vert`/`vrt2` の直立グリフ形が選ばれ、フォントの縦書きメトリクス（`vmtx`/`VORG`）を使ってグリフが上から下へ積み重ねられます。
- **リガチャ、マークの配置、カーニング** — すべてのスクリプトの経路で、GSUB の `ccmp`/`rlig`/`liga`/`calt` に続いて GPOS の `kern`/`mark`/`mkmk`/`curs` を適用します。これにより、ダイアクリティカルマークはベース文字の上に正しく載り、文字ペアのカーニングも行われます。

## 使い方 {#usage}

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

基本の方向はデフォルトで `Auto`（最初の強方向文字から決定）となり、スクリプトはテキストから自動検出されます（アラビア文字ブロックのルーンがあればアラビア語が、インド系や USE のルーンがあればそれぞれのシェーパが選ばれます）。ただし、`Options.Script` または `Options.Direction` を設定した場合は、その指定が優先されます。

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

## 対応範囲 {#scope}

**実装済み:** アラビア語、専用シェーパを持つ 10 の**インド系**スクリプト、**Universal Shaping Engine**（タイ語、ラオ語、クメール語、ミャンマー語、チベット語、ジャワ語、バリ語、ブギス語、タイタム文字など）、**エジプト聖刻文字**のクワドラットレイアウト、**ハングル**のジャモ合成、**縦書き**モード、そして **Latin/default**（ラテン文字、キリル文字、ギリシャ文字、横書きの CJK など）のシェーピングです。

クラスタのインデックスは、1 対 1 の置換（アラビア文字の位置形）では正確です。置換によって連なりの長さが変わる場合（リガチャ、分解、インド系/USE の並べ替え）は、単調性を保つベストエフォートです。

ソース: [github.com/go-opentype/shape](https://github.com/go-opentype/shape) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/shape)
