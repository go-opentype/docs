---
title: "opentype"
weight: 20
---

# opentype — エンジン本体 {#opentype-the-engine}

`github.com/go-opentype/opentype` は、純粋な Go 製、`CGO_ENABLED=0`、**標準ライブラリのみ**で実装された、TrueType/OpenType フォントのパーサとアンチエイリアス付きラスタライザ（rasteriser）です。インポートするのは `image`、`encoding/binary`、`math`、`errors` などだけで、`golang.org/x/*` もサードパーティのモジュールも使いません。そのため、`GOOS=js GOARCH=wasm` を含むすべての Go ターゲットでビルドできます。

グリフ描画型の UI が必要とする `golang.org/x/image/font/opentype` の限られた一部、つまりフォントの blob の解析、ピクセルサイズでのフェイスの構築、ルーンごとのアドバンス幅と 8 ビットのアルファカバレッジマスクの取得を機能面で置き換えるために作られており、さらにその先へ進みます。CFF/CFF2 アウトライン、可変フォント、GPOS/GSUB シェーピング、カーニング（kerning）、TrueType/CFF ヒンティング（hinting）もすべて対象範囲です。

## 使い方 {#usage}

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

`GlyphMask` の形は、意図的に `x/image` の `font.Face.Glyph` に合わせてあるため、x/image のフェイスからの置き換えは機械的に行えます。

## 可変フォント {#variable-fonts}

`SetVariation` を使うと、可変フォントのデザイン空間の特定の点でインスタンスを作れます。軸はタグ（`wght`、`wdth`、`opsz` など）で指定します。

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

`SetVariation` は、グリフのアウトライン（TrueType では `gvar`、CFF2 では CFF2 のブレンド）を再補間し、選んだ座標に対して `HVAR`/`VVAR`/`MVAR` によるメトリクスを再計算します。`avar` による軸値のマッピングは自動的に適用されます。

## ヒンティング {#hinting}

`SetHinting(true)` は、ラスタライズの前に、TrueType 命令インタープリタ（`glyf` アウトライン用）または CFF/Type 2 のステムグリッドフィッタ（CFF/CFF2 アウトライン用）を実行します。`SetStemDarkening(true)` は、これに加えて CFF のステムダークニングを適用します。どちらもデフォルトでは無効で、その場合は一様な 4×4 スーパーサンプリングで描画されます。

## 数式組版（`MATH` テーブル） {#mathematical-typesetting-the-math-table}

`opentype` は OpenType の **`MATH`** テーブルをデコードします。これは、数式組版エンジン（LuaTeX、unicode-math、MathJax、Word の数式エディタ、あるいは HarfBuzz の `hb-ot-math` を土台にしたもの）が数式をレイアウトするために必要とする、フォント単位のメトリクスです。3 つの部分から成り、いずれも `Face` を通じてピクセル単位にスケールされた値として公開されます。

- **MathConstants** — 約 56 個のグローバルなスカラー値です。数式軸の高さ、下付き・上付き文字のシフトとギャップの最小値、分数と根号のギャップ、罫線の太さなどがあり、`MathConstant` と、`opentype.AxisHeight` や `opentype.FractionRuleThickness` のような `MathConstant` の値で読み出します（ディスク上の順序での全一覧は `go doc` を参照してください）。
- **MathGlyphInfo** — グリフごとの*イタリック補正*（`ItalicCorrection`。斜体のグリフの後に直立のグリフを置く前に挿入するスペース、または上付き文字の後に続く下付き文字の水平オフセット）、*トップアクセント位置*（`TopAccentAttachment`。グリフの上にアクセントを中央揃えにする x 位置）、*拡張形状*フラグ（`IsExtendedShapeGlyph`。大きな積分記号のように、グリフが引き伸ばされているかのように上付き文字を配置するグリフ用）、そして 4 隅の切り込み式*数式カーニング*（`MathKern`。隣接する添え字の取り付け高さに対する階段関数）です。
- **MathVariants** — グリフのより大きなサイズのバリアントと、最大のバリアントでも足りない場合に、任意の高さや幅の伸縮グリフ（積分記号、波括弧、根号、矢印）を、繰り返し可能な部品から組み立てるためのレシピです。どちらも `MathVariants` がまとめて返します。

数式の*レイアウト*そのもの、つまりボックスモデル、TeX の数式リストの規則、バリアントの選択、ターゲットサイズまでのアセンブリの拡大は、これらのメトリクスの上に構築される上位レベルの関心事です。`hb-ot-math` がメトリクスだけを公開してレイアウトを呼び出し側に任せているのと同じく、このパッケージの対象外です。

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

実際の STIX Two Math フォントを 1 em あたり 32px で実行すると、`axis height: 8 fraction rule thickness: 2`、`italic correction: 1`、`size variants: 13 assembly parts: 3` と出力されます。STIX Two Math（SIL Open Font License）は、`opentype` リポジトリの [`testdata/STIXTwoMath-Regular.otf`](https://github.com/go-opentype/opentype/blob/main/testdata/STIXTwoMath-Regular.otf) として同梱されており、デコーダのブランチカバレッジを 100% にする合成 MATH テーブルのフィクスチャとあわせて、CI でエンドツーエンドに検証されています。

## フォントディスクリプタとサブセット化 {#font-descriptors-and-subsetting}

フォントを*埋め込む*プログラム、たとえば PDF ライターや CSS の `@font-face` ジェネレータのために、`opentype` は書き出し側のプリミティブも公開しています。

- **ディスクリプタのスカラー値** — OS/2 と `head`/`post` から得られ、PDF の `/FontDescriptor` や CSS の宣言にそのまま使えます。`Ascent`、`Descent`、`LineGap`、`FontBBox`、`CapHeight`、`XHeight`、`ItalicAngle`、`WeightClass`、`WidthClass`、`StemV`、`Flags`、および真偽値の `IsFixedPitch`/`IsItalic`/`IsSerif` です。
- **インスタンス化** — `Instance(coords)` は、可変フォントのデザイン空間の 1 点に固定した（glyf/CFF2 のアウトラインとメトリクスを凍結した）新しい `*Font` を返し、`InstanceBytes(coords)` はそのインスタンスを新しい sfnt の blob として返します。
- **サブセット化** — `SubsetTrueType(gids)` は、TrueType フォントを要求されたグリフだけに絞り込みます。複合グリフの依存関係を閉包し、旧から新へのグリフ ID の対応表を返します。`SubsetCFF(gids)` は CFF/OTTO フォントに対して同じことを行い、`seac` のアクセント参照を閉包します。どちらも、ドキュメントが実際に使うグリフだけを残すので、埋め込まれたフォントは小さく保たれます。

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

## WOFF2 とバイナリサイズ {#woff2-and-binary-size}

**状況: main に取り込み済みで、次のリリースに含まれます**（マージ済みですが、まだタグは付いていません）。

WOFF2 フォントの展開には、`github.com/andybalholm/brotli` ではなく、デコーダのみの `github.com/dsnet/compress/brotli` を使うようになりました。`andybalholm/brotli` のルートパッケージは `net/http` をインポートしていたため、`opentype` をインポートするすべてのプログラムに `net/http` と `crypto/tls` の `init` がリンクされていました。`GOOS=js` 向けの 7.5 MB のテキストプログラムのうち 1.8 MB がこれに当たります。デコーダのみのパッケージは、そのどちらもインポートしません。

展開は、WOFF2 のディレクトリが宣言するサイズ（最大 256 MiB）で上限が設けられるようになり、展開爆弾（decompression bomb）はメモリを確保されずに拒否されます。

未解決の項目（修正ではなくトレードオフ）: `dsnet/compress/brotli` は展開が約 1.37 倍遅く、200 KB のフォントで約 0.4 ms です。サイズと上限の効果のために受け入れており、後で見直せるようここに記録しています。

## API {#api}

| Symbol | Purpose |
| --- | --- |
| `Parse(b []byte) (*Font, error)` | sfnt（TrueType または CFF/OTTO）のフォント blob をデコードします。 |
| `(*Font).NumGlyphs() int` | グリフ数（`maxp` から）です。 |
| `(*Font).GlyphIndex(r rune) (GlyphIndex, bool)` | cmap でルーンをグリフに対応付けます。 |
| `(*Font).GlyphIndexVariation(r, vs rune) (GlyphIndex, bool)` | ルーンと Unicode の異体字セレクタを対応付けます。 |
| `(*Font).Axes() []Axis` | 可変フォントのデザイン軸（`fvar`）です。 |
| `(*Font).NamedInstances() []NamedInstance` | バリエーション空間内の名前付きの位置です。 |
| `(*Font).GPOS() *GPOS` / `(*Font).GSUB() *GSUB` | 解析済みのレイアウトテーブル、または `nil` です。 |
| `(*Font).HasVerticalMetrics() bool` | `vhea`/`vmtx` があるかどうかです。 |
| `(*Font).NewFace(sizePx int) *Face` | サイズ指定済みのフェイスを構築します（スケール = `sizePx / unitsPerEm`）。 |
| `(*Face).Metrics() Metrics` | ピクセル単位の `Ascent`、`Descent`、`Height` です。 |
| `(*Face).Advance(r rune) int` / `(*Face).Measure(s string) int` | ルーン／文字列のアドバンス幅（ピクセル単位）です。 |
| `(*Face).Kern(prev, r rune) int` / `(*Face).MeasureKerned(s string) int` | GPOS または `kern` によるペア調整です。 |
| `(*Face).GlyphMask(r, x, y) (image.Rectangle, *image.Alpha, image.Point, int, bool)` | ラスタライズされたグリフとアドバンス幅です。 |
| `(*Face).Shape(text string, features ...string) []GlyphIndex` | GSUB で置換されたグリフ列です。 |
| `(*Face).ShapePositioned(text string, features ...string) []PositionedGlyph` | ペンのオフセット付きの、GSUB + GPOS を適用したグリフ列です。 |
| `(*Face).SetVariation(coords map[string]float64)` | 指定した軸の座標で可変フォントのインスタンスを作ります。 |
| `(*Face).SetHinting(on bool)` / `(*Face).SetStemDarkening(on bool)` | TrueType/CFF ヒンタを切り替えます。 |
| `(*Face).VerticalAdvance(r rune) int` / `(*Face).VerticalOrigin(r rune) (int, bool)` | 縦書きモードのメトリクスです。 |
| `(*Font).HasMath() bool` | フォントに OpenType の `MATH` テーブルがあるかどうかです。 |
| `(*Font).IsExtendedShapeGlyph(gid GlyphIndex) bool` | gid が `MATH` の拡張形状カバレッジに含まれるかどうかです。 |
| `(*Face).MathConstant(which MathConstant) int` | ピクセル単位にスケールされた約 56 個の `MATH` レイアウト定数（数式軸の高さ、添え字のシフト、分数・根号のギャップなど）のうちの 1 つです。 |
| `(*Face).ItalicCorrection(gid GlyphIndex) int` | グリフごとの数式イタリック補正（ピクセル単位）です。 |
| `(*Face).TopAccentAttachment(gid GlyphIndex) (int, bool)` | gid の上にトップアクセントを中央揃えにする X 位置（ピクセル単位）です。 |
| `(*Face).MathKern(gid GlyphIndex, corner MathKernCorner, correctionHeight int) int` | 指定した取り付け高さでの、gid の 4 隅のうち 1 つに対する切り込み式の数式カーニング（ピクセル単位）です。 |
| `(*Face).MathVariants(gid GlyphIndex, vertical bool) ([]MathVariant, *MathAssembly)` | 1 つの軸に沿った、gid のサイズバリアントと伸縮グリフのアセンブリです。 |
| `(*Font).Ascent/Descent/LineGap/FontBBox/CapHeight/XHeight/ItalicAngle/WeightClass/WidthClass/StemV/Flags/IsFixedPitch/IsItalic/IsSerif` | PDF の `/FontDescriptor` と CSS の `@font-face` のための、OS/2 + `head`/`post` から得られるフォントディスクリプタのスカラー値です。 |
| `(*Font).Instance(coords map[string]float64) (*Font, error)` / `(*Font).InstanceBytes(coords) ([]byte, error)` | 可変フォントを静的インスタンス（`*Font` または新しい sfnt blob）に固定します。 |
| `(*Font).SubsetTrueType(gids []GlyphIndex) ([]byte, map[GlyphIndex]GlyphIndex, error)` | TrueType フォントをグリフ集合にサブセット化し、複合グリフを閉包します。旧から新への ID 対応表を返します。 |
| `(*Font).SubsetCFF(gids []GlyphIndex) ([]byte, error)` | CFF/OTTO フォントをグリフ集合にサブセット化し、`seac` のアクセントを閉包します。 |

`Font` は `Parse` の後は不変で、並行利用しても安全です。`Face` はラスタライズしたグリフをキャッシュし、並行利用しても安全では**ありません**。必要なら goroutine ごとに `Face` を 1 つ作ってください。

## 対応表 {#support-matrix}

- sfnt コンテナ（`0x00010000` と `true` の TrueType マジック、`OTTO` の CFF マジック）
- `head`、`maxp`、`hhea`、`hmtx`（末尾の共有アドバンスの連続を含む）
- `cmap` のフォーマット **0、2、4、6、8、10、12、13**、そして **14**（Unicode 異体字シーケンス）。12 を優先します
- `loca`（ショートとロング）、`glyf` の単純グリフ（繰り返しフラグ、短い/長いデルタ）と複合グリフ（`ARGS_ARE_XY_VALUES`、スケール / x と y のスケール / 2×2、`MORE_COMPONENTS`。循環と深さのガード付き）
- 2 次曲線の輪郭に対する、暗黙のオンカーブ中間点の合成
- CFF と CFF2（`OTTO`）の Type 2 チャーストリングのアウトライン。サブルーチン、`seac` によるアクセント合成、CFF2 の可変フォント用ブレンドオペレータを含みます
- `fvar` の軸と名前付きインスタンス、`avar` による軸値のマッピング、`gvar`/CFF2 によるグリフアウトラインの補間、`HVAR`/`VVAR`/`MVAR` によるメトリクスのバリエーション。いずれも `(*Face).SetVariation` を通じて利用します
- すべての `GSUB`/`GPOS` ルックアップタイプ（リガチャ（ligature）、文脈依存置換と位置形置換（`isol`/`init`/`medi`/`fina`）、単一/ペア/連続の位置調整、mark-to-base と mark-to-mark の配置）に加え、`GDEF` とルックアップフラグによるマークのフィルタリング
- GPOS のペア位置調整によるカーニングと、従来の `kern` テーブルへのフォールバック。単一の `Kerner` を通じて提供されます
- TrueType 命令ヒンタと、CFF/Type 2 のステムグリッドフィッタ（ブルーゾーン、カウンタ、ステムダークニング）。`Face` ごとに `SetHinting` / `SetStemDarkening` で切り替えます
- 縦書きモード向けの、`vhea`/`vmtx`/`VORG` に対応した縦書きメトリクスと原点
- OpenType の **`MATH`** テーブル — 数式組版のメトリクス（定数、グリフごとのイタリック補正 / トップアクセント位置 / 4 隅のカーニング、伸縮グリフのサイズバリアントとアセンブリ）を、`Face` を通じてピクセル単位にスケールして公開します（`HasMath`、`MathConstant`、`ItalicCorrection`、`TopAccentAttachment`、`MathKern`、`MathVariants`、`IsExtendedShapeGlyph`）。数式の*レイアウト*は上位レベルのエンジンに任されています
- OS/2 + `head`/`post` の**フォントディスクリプタ**のスカラー値（`Ascent`、`Descent`、`FontBBox`、`CapHeight`、`XHeight`、`ItalicAngle`、`WeightClass`、`WidthClass`、`StemV`、`Flags` など）。PDF や CSS への埋め込み用です
- 可変フォントの静的マスターへの**インスタンス化**（`Instance`、`InstanceBytes`）と、グリフ集合への**サブセット化**（複合グリフの閉包と旧から新への ID 対応表を伴う `SubsetTrueType`、`seac` の閉包を伴う `SubsetCFF`）
- 非ゼロの巻き数規則のもとでの、4×4 スーパーサンプリングによるアンチエイリアス付きラスタライズ

ラスタライズは一様なスーパーサンプリングを用いており、ネイティブの TrueType/CFF ヒンタのドロップアウト制御に基づくデルタヒンティング付きのラスタライズではありません。そのため、小さいサイズでの最後の一滴のシャープさよりも、正確さと移植性を優先しています。これは、HarfBuzz + FreeType がカバーする範囲において機能的に完全なエンジンであり、上に挙げたものにスタブはありません。

## テスト {#testing}

テストは、100% のカバレッジに到達するために外部のフォントに依存することはありません。最小限でありながら有効な TrueType と CFF のフォント（テーブルディレクトリ + `head`/`maxp`/`hhea`/`hmtx`/`cmap`/`loca`/`glyf`、または合成の `MATH` テーブル）をメモリ上で合成し、エラー経路も含めてすべての解析とラスタライズの分岐を決定論的に実行します。実在のフォント、すなわち Adobe の Source Serif 4 と、`MATH` テーブル用の STIX Two Math（どちらも SIL Open Font License で、`testdata/` 以下に同梱）も、`example_test.go` といくつかの健全性チェックのテストでエンドツーエンドに実行されます。そのため、ドキュメントに載っている例は、実運用のフォントデータに対するスモークテストを兼ねています。CI では、**100.0% のステートメントカバレッジ**、`go vet`、そして `linux/{amd64,arm64,riscv64,loong64,ppc64le,s390x}`、`js/wasm`、`darwin/arm64`、`windows/amd64` へのクロスコンパイルのスモークテストを必須としています。

ソース: [github.com/go-opentype/opentype](https://github.com/go-opentype/opentype) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/opentype)
