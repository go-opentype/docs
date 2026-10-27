---
title: "选择无障碍字体"
weight: 90
---

# 指南：选择无障碍字体 {#guide-choose-an-accessible-font}

[`fonts`]({{< relref "../fonts.md" >}}) 内置了 46 个字体族，使 Go 程序无需寻找或下载 `.ttf`，就能附带易读的文本。本指南讲的是如何挑选合适的字体，而不仅仅是使用默认字体。

## 只想要易读的字体？使用默认值 {#just-want-something-legible-use-the-default}

```go
f, err := fonts.Parse(fonts.MostLegible())
```

`MostLegible()` 返回 **Atkinson Hyperlegible**，由美国盲文研究所（Braille Institute of America）专为最大限度区分字符而设计，面向低视力读者：在大多数无衬线字体中最容易混淆的字形（`I`/`l`/`1`、`O`/`0`）被有意做了区分。这对所有读者都有益，而不只是低视力读者，这也是它成为包默认值而不是可选项的原因。

## 按用途挑选 {#picking-by-role}

`fonts.All()` 返回每一个内置的 `Family`，每个都带有 `Kind` 标签（`KindSans`、`KindSerif`、`KindMono`），这样你可以按用途筛选，而不必硬编码名称：

```go
for _, fam := range fonts.All() {
    if fam.Kind == fonts.KindMono {
        // e.g. Go Mono or JetBrains Mono — a code block face
    }
}
```

或者按名称查找，以获知它的导入路径（`Family` 不带 `[]byte`：枚举或搜索字体族绝不会把它们的字节链接进来；要获取字节本身，你仍需导入对应的子包）：

```go
fam, ok := fonts.ByName("Lora") // case-insensitive
if ok {
    fmt.Println(fam.ImportPath) // github.com/go-opentype/fonts/lora
}
// import "github.com/go-opentype/fonts/lora", then:
f, err := fonts.Parse(lora.TTF)
```

## 内置了什么，以及为什么选它 {#whats-bundled-and-why-youd-pick-it}

| 字体族 | 类别 | 最适合 |
| --- | --- | --- |
| **Atkinson Hyperlegible** | Sans | 默认选择；最看重易读性的正文 |
| Inter | Sans | 界面外观、信息密集的界面，更中性的几何无衬线体 |
| Go | Sans | Go 项目自己的无衬线体，是 Go 工具类界面的自然之选 |
| Lora | Serif | 较长篇幅的阅读、编辑类场景 |
| Go Mono | Mono | 代码，与上面的 Go 无衬线体相配 |
| JetBrains Mono | Mono | 代码，更宽、对连字友好的等宽字体 |

## 可变字体的注意事项 {#the-variable-font-caveat}

Inter、Lora 和 JetBrains Mono 在上游是**可变字体**（单个文件涵盖一段字重/字宽轴范围）。`go-opentype/fonts` 把每一个都固定在它的**默认静态实例**上：这里内置的 `.ttf` 是某一个固定字重，而不是完整的可变范围，这是为了保持打包简单、体积小，而不是因为引擎无法处理轴：[`opentype` 完整支持 OpenType 可变字体]({{< relref "../opentype.md#variable-fonts" >}})，包括用 `SetVariation` 实例化任意轴坐标。如果你的设计需要沿某个轴取得特定字重（例如字重 600 的 Inter，而不是默认的 400），请自行获取该字体族的可变 `.ttf`，例如从 [google/fonts](https://github.com/google/fonts) 获取，并用 `opentype.Parse` + `(*Face).SetVariation(map[string]float64{"wght": 600})` 来驱动它。

## 许可证义务 {#license-obligations}

`fonts` 中的 Go 源代码采用 BSD-3-Clause，但**字体文件本身**保留其上游许可证：Atkinson Hyperlegible、Inter、Lora 和 JetBrains Mono 为 OFL-1.1；Go/Go Mono 为 BSD-3-Clause。如果你分发嵌入了这些字体的二进制文件，请随附 `licenses/` 中对应的许可证文本；完整的表格和版权声明请参见 [fonts]({{< relref "../fonts.md#bundled-fonts" >}})。
