---
title: "bidi"
weight: 30
---

# bidi：Unicode 双向算法 {#bidi-the-unicode-bidirectional-algorithm}

`github.com/go-typeset/bidi` 是 [Unicode 双向算法（UAX #9）](https://www.unicode.org/reports/tr9/)的纯 Go 实现，使用 `CGO_ENABLED=0` 构建，**仅依赖标准库**，用于排布左到右/右到左混合文本。

与大多数 Go 的 bidi 实现不同，它**不**依赖 `golang.org/x/text`。`Bidi_Class` 和成对括号（paired-bracket）这两项 Unicode 属性被编译成小型的生成式查找表（参见 [`cmd/genbidi`](https://github.com/go-typeset/bidi/tree/main/cmd/genbidi)），因此只要标准库能构建的地方，这个包就能构建。

{{< callout type="warning" >}}
**它已迁移，旧的导入路径会失败**


这个包原先是 `github.com/go-opentype/bidi`。迁移时保留了一个重定向，所以旧的**链接**仍然可以打开，但旧的**导入路径**无法使用，因为该模块声明的是新路径：

```
$ go get github.com/go-opentype/bidi@latest
module declares its path as: github.com/go-typeset/bidi
        but was required as: github.com/go-opentype/bidi
```
{{< /callout >}}

## 用法 {#usage}

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

| 符号 | 用途 |
| --- | --- |
| `ClassOf(r rune) Class` | 一个 rune 的 Bidi_Class |
| `Class` enum | `L R AL EN ES ET AN CS NSM BN B S WS ON LRE RLE LRO RLO PDF LRI RLI FSI PDI` |
| `ResolveLevels(text []rune, base Direction) []Level` | 每个 rune 解析后的嵌入层级 |
| `BaseLevel(text []rune, base Direction) Level` | 段落层级（规则 P2/P3） |
| `Reorder(text []rune, levels []Level) []int` | 规则 L2 的视觉顺序排列 |
| `VisualOrder(text string, base Direction) string` | 解析并重排的便捷函数 |
| `Direction` | `LeftToRight`、`RightToLeft`、`Auto` |
| `Level` | 嵌入层级（偶数 = LTR，奇数 = RTL） |

## 已实现与已推迟 {#implemented-vs-deferred}

**已实现**：算法运行到规则 **L2**，这正是 Unicode 一致性测试文件 `BidiCharacterTest.txt` 所覆盖的全部范围：

- **P2、P3**：根据第一个强方向字符确定基础段落层级。
- **X1–X8**：显式嵌入*以及*隔离（含溢出处理和方向状态栈）；**X9**：移除已弃用的格式字符和 `BN`；**X10**：带有 `sos`/`eos` 的隔离运行序列。
- **W1–W7**：弱类型。
- **N0**：成对括号解析（BD16，包括 U+2329/U+232A 的规范等价），**N1–N2**：中性类型。
- **I1、I2**：隐式层级。
- **L1**：分隔符/行尾空白重置，**L2**：重排。

**已推迟**（超出 bidi 引擎的范围，属于整形器的工作）：

- **L3**（组合标记）和 **L4**（成对括号及其他镜像字符的字形镜像）：`VisualOrder` 不会替换镜像字形，那属于渲染/整形阶段。
- 阿拉伯文连写的**整形/连接**：参见 [shape]({{< relref "shape.md" >}})。
- **P1** 段落拆分：由调用方负责；该 API 按段落运行（行内的 `Paragraph_Separator` 仍由 X8/L1 处理）。

## 一致性 {#conformance}

该包已针对**完整的** `BidiCharacterTest.txt` 进行验证（所有用例均通过：段落层级、逐字符层级和视觉顺序）。一个经过挑选的代表性子集嵌入在 `testdata` 下，由 `TestConformance` 运行。CI 强制要求语句覆盖率**恰好为 100%**，并执行 `go vet`、`gofmt`，以及针对六种 64 位架构外加 `js/wasm`、`darwin/arm64` 和 `windows/amd64` 的交叉编译。

## 重新生成表 {#regenerating-the-tables}

```sh
go run ./cmd/genbidi .
```

它会从 Unicode 字符数据库获取最新的 `DerivedBidiClass.txt` 和 `BidiBrackets.txt`，并重写 `bidiclass_table.go` 和 `bidibrackets_table.go`。

源代码：[github.com/go-typeset/bidi](https://github.com/go-typeset/bidi) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-typeset/bidi)
