---
title: "アクセシブルなフォントを選ぶ"
weight: 90
---

# ガイド: アクセシブルなフォントを選ぶ {#guide-choose-an-accessible-font}

[`fonts`]({{< relref "../fonts.md" >}}) には 46 のファミリーが同梱されており、Go プログラムは `.ttf` を調達したりダウンロードしたりせずに、読みやすいテキストを出荷できます。このガイドでは、デフォルトだけでなく、用途に合ったフォントの選び方を説明します。

## 読みやすければよいなら、デフォルトを使う {#just-want-something-legible-use-the-default}

```go
f, err := fonts.Parse(fonts.MostLegible())
```

`MostLegible()` は **Atkinson Hyperlegible** を返します。これは Braille Institute of America が、弱視の読者にとって文字の区別がつきやすくなるよう特に設計したフォントで、ほとんどのサンセリフで紛らわしい字形（`I`/`l`/`1`、`O`/`0`）が、意図的に見分けやすくされています。この利点は弱視の読者に限らずすべての読者に及ぶため、オプトインではなくパッケージのデフォルトになっています。

## 役割で選ぶ {#picking-by-role}

`fonts.All()` は同梱されているすべての `Family` を返し、それぞれに `Kind`（`KindSans`、`KindSerif`、`KindMono`）が付いています。名前を直接書き込まなくても、役割で絞り込めます。

```go
for _, fam := range fonts.All() {
    if fam.Kind == fonts.KindMono {
        // e.g. Go Mono or JetBrains Mono — a code block face
    }
}
```

また、名前で検索してインポートパスを調べることもできます（`Family` は `[]byte` を持たないため、ファミリーの列挙や検索でバイト列がリンクされることはありません。バイト列そのものを使うには、やはりサブパッケージをインポートします）。

```go
fam, ok := fonts.ByName("Lora") // case-insensitive
if ok {
    fmt.Println(fam.ImportPath) // github.com/go-opentype/fonts/lora
}
// import "github.com/go-opentype/fonts/lora", then:
f, err := fonts.Parse(lora.TTF)
```

## 同梱フォントと選ぶ理由 {#whats-bundled-and-why-youd-pick-it}

| Family | Kind | Best for |
| --- | --- | --- |
| **Atkinson Hyperlegible** | Sans | 既定の選択肢です。読みやすさが最も重要な本文向けです |
| Inter | Sans | UI のクローム（枠や操作部品）や情報密度の高いインターフェース向けです。より中立的なジオメトリック系サンセリフです |
| Go | Sans | Go プロジェクト自身のサンセリフです。Go のツール類の UI に自然な選択です |
| Lora | Serif | 長文の読み物や編集向けの用途に適しています |
| Go Mono | Mono | コード向けです。上の Go サンセリフと揃えられます |
| JetBrains Mono | Mono | コード向けです。リガチャ（ligature）に向いた、幅の広い等幅フォントです |

## 可変フォントに関する注意点 {#the-variable-font-caveat}

Inter、Lora、JetBrains Mono は、上流では**可変フォント**として提供されています（ウェイトや幅の軸の範囲全体を 1 つのファイルでまかないます）。`go-opentype/fonts` は、バンドルを単純かつ小さく保つために、それぞれを**デフォルトの静的インスタンス**に固定しています。ここに同梱されている `.ttf` は可変の範囲全体ではなく、特定の 1 つのウェイトです。これはエンジンが軸を扱えないからではありません。[`opentype` は OpenType Variations に完全に対応しており]({{< relref "../opentype.md#variable-fonts" >}})、`SetVariation` で任意の軸の値のインスタンスを作れます。デザイン上、軸に沿った特定のウェイト（たとえば Inter のデフォルトの 400 ではなく 600）が必要な場合は、そのファミリーの可変 `.ttf` を（たとえば [google/fonts](https://github.com/google/fonts) から）自分で入手し、`opentype.Parse` と `(*Face).SetVariation(map[string]float64{"wght": 600})` で操作してください。

## ライセンス上の義務 {#license-obligations}

`fonts` の Go ソースは BSD-3-Clause ですが、**フォントファイル自体**は上流のライセンスのままです。Atkinson Hyperlegible、Inter、Lora、JetBrains Mono は OFL-1.1、Go と Go Mono は BSD-3-Clause です。これらのフォントを埋め込んだバイナリを再配布する場合は、`licenses/` にある対応するライセンス文を一緒に配布してください。全一覧と著作権表示は [fonts]({{< relref "../fonts.md#bundled-fonts" >}}) を参照してください。
