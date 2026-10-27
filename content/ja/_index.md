---
title: "概要"
weight: 10
---

# go-opentype ドキュメント {#go-opentype-documentation}

**依存ライブラリなしの、純粋な Go 製 TrueType/OpenType テキストスタックです。** 4 つの小さなモジュールが連携して、フォントファイルの読み込み、Unicode の双方向テキストと複雑なスクリプトの処理、結果のラスタライズ（rasterisation）までを行います。スタック全体で `golang.org/x/*` パッケージも cgo も一切使いません。

{{< callout type="info" >}}
**サードパーティへの依存はゼロ**

スタック内のすべてのモジュールは Go の標準ライブラリ（`image`、`encoding/binary`、`math`、`errors` など）だけをインポートします。そのため、C ツールチェーンも同梱アセットも不要で、`GOOS=js GOARCH=wasm` を含むあらゆる Go ターゲットへ簡単にクロスコンパイルできます。
{{< /callout >}}

## モジュール一覧 {#the-modules}

| Repo | Module path | What it is |
| --- | --- | --- |
| [`opentype`]({{< relref "opentype.md" >}}) | `github.com/go-opentype/opentype` | エンジン本体です。sfnt の解析とアンチエイリアス付きラスタライズを担います |
| [`bidi`]({{< relref "bidi.md" >}}) | `github.com/go-opentype/bidi` | Unicode 双方向アルゴリズム（UAX #9）です |
| [`shape`]({{< relref "shape.md" >}}) | `github.com/go-opentype/shape` | 上記 2 つを組み合わせる、HarfBuzz-lite な複雑テキストシェーパです |
| [`fonts`]({{< relref "fonts.md" >}}) | `github.com/go-opentype/fonts` | 読みやすさを重視した 46 のフォントファミリーを、ファミリーごとに `go:embed` で同梱します |
| [`docs`](https://github.com/go-opentype/docs) | — | このドキュメントサイトです（MkDocs Material、mike でバージョン管理） |
| [`brand`](https://github.com/go-opentype/brand) | — | ロゴとブランドアセットです |

各モジュールは独立しており、単独でインポートできます。兄弟モジュールに依存するのは `shape` だけで、`opentype`、`bidi`、`fonts` は組織内の他モジュールに依存しません（`fonts` が依存するのは、`Parse` の便利ラッパーで使う `opentype` の `*Font` 型だけです）。

## 置き換えの対象 {#what-it-replaces}

`go-opentype` は、グリフ（glyph）をピクセルバッファへ描画する Go プログラムにとって、次のものを機能面で置き換えるために作られています。

- グリフ描画型の UI が必要とする、`golang.org/x/image/font` と `font/opentype` の限られた一部（フォントの解析、サイズ指定済みフェイスの構築、アドバンス幅とカバレッジマスクの取得）。
- 左から右（LTR）と右から左（RTL）が混在するテキストをレイアウトするための `golang.org/x/text/unicode/bidi`。

このスタックは HarfBuzz と FreeType の組み合わせとの機能的な同等性を目標としており、対応する範囲では現時点でそれを達成しています。対応しているのは、TrueType の `glyf` **および** CFF/CFF2 アウトラインの解析とラスタライズ、すべての GSUB/GPOS ルックアップタイプと GDEF によるマークのフィルタリング、OpenType Variations（`fvar`/`avar`/`gvar`/`HVAR`/`VVAR`/`MVAR`）、TrueType 命令ヒンタ（hinter）と CFF/Type 2 のステムグリッドフィッタ、縦書きメトリクス、数式組版のための OpenType `MATH` テーブル（HarfBuzz の `hb-ot-math` に対応）、規則 L2 までの UAX #9 双方向アルゴリズムの全体、そしてアラビア語、インド系文字、Universal Shaping Engine（タイ語、ラオ語、クメール語、ミャンマー語、チベット語など）、エジプト聖刻文字のクワドラット、ハングル、縦書き（CJK の tategaki）レイアウト、Latin/default の複雑テキストシェーピングです。各モジュールの対応表は、それぞれのページを参照してください。

## インストール {#install}

```sh
go get github.com/go-opentype/opentype
go get github.com/go-opentype/bidi
go get github.com/go-opentype/shape
go get github.com/go-opentype/fonts
```

## 次に読むページ {#where-to-go-next}

- [opentype]({{< relref "opentype.md" >}}) — sfnt パーサとラスタライザです。フォント、`cmap`、TrueType の `glyf` と CFF/CFF2 アウトライン、可変フォント、GSUB/GPOS シェーピング、ヒンティング（hinting）、縦書きメトリクス、数式組版のための `MATH` テーブル、フォントのサブセット化とディスクリプタのアクセサ、完全な対応表を扱います。
- [bidi]({{< relref "bidi.md" >}}) — UAX #9 エンジンです。何を実装し、何をシェーパに委ねているかを説明します。
- [shape]({{< relref "shape.md" >}}) — 複雑テキストシェーパです。アラビア文字の連綿結合、インド系文字の並べ替え、Universal Shaping Engine、エジプト聖刻文字のクワドラット、ハングル、縦書きレイアウト、リガチャ（ligature）、マークの配置、カーニング（kerning）、対応するスクリプトを扱います。
- [fonts]({{< relref "fonts.md" >}}) — 同梱の 46 のファミリーと、それらを使うためのファミリーごとのインポート方式を説明します。
- **ガイド** — 目的別の手順解説です。テキストを PNG に描画する方法、方向が混在するアラビア語文字列のシェーピング、読みやすさを基準にした同梱フォントの選び方を扱います。

ソースコードは [github.com/go-opentype](https://github.com/go-opentype) にあります。
