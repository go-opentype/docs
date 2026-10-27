---
title: "bidi"
weight: 30
---

# bidi — 유니코드 양방향 알고리즘 {#bidi-the-unicode-bidirectional-algorithm}

`github.com/go-typeset/bidi`는 왼쪽에서 오른쪽, 오른쪽에서 왼쪽 텍스트가 섞인 문장을 배치하기 위한 [유니코드 양방향 알고리즘(Unicode Bidirectional Algorithm, UAX #9)](https://www.unicode.org/reports/tr9/)의 순수 Go, `CGO_ENABLED=0`, **표준 라이브러리 전용** 구현입니다.

다른 대부분의 Go bidi 구현과 달리 `golang.org/x/text`에 의존하지 **않습니다**. `Bidi_Class`와 짝 괄호 유니코드 속성은 생성된 작은 조회 테이블로 컴파일되어 있으므로([`cmd/genbidi`](https://github.com/go-typeset/bidi/tree/main/cmd/genbidi) 참고), 표준 라이브러리가 빌드되는 곳이라면 어디서든 이 패키지를 빌드할 수 있습니다.

{{< callout type="warning" >}}
**이전되었으며, 이전 임포트 경로는 실패합니다**


이 패키지는 예전에 `github.com/go-opentype/bidi`였습니다. 이전 과정에서 리디렉션이 남아 있어 이전 **링크**는 여전히 열리지만, 모듈이 새 경로를 선언하고 있으므로 이전 **임포트**는 동작하지 않습니다.

```
$ go get github.com/go-opentype/bidi@latest
module declares its path as: github.com/go-typeset/bidi
        but was required as: github.com/go-opentype/bidi
```
{{< /callout >}}

## 사용법 {#usage}

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
| `ClassOf(r rune) Class` | 룬의 Bidi_Class |
| `Class` enum | `L R AL EN ES ET AN CS NSM BN B S WS ON LRE RLE LRO RLO PDF LRI RLI FSI PDI` |
| `ResolveLevels(text []rune, base Direction) []Level` | 룬별로 결정된 임베딩 레벨 |
| `BaseLevel(text []rune, base Direction) Level` | 문단 레벨 (규칙 P2/P3) |
| `Reorder(text []rune, levels []Level) []int` | 규칙 L2의 시각적 순서 순열 |
| `VisualOrder(text string, base Direction) string` | 결정 + 재배열을 한 번에 하는 편의 함수 |
| `Direction` | `LeftToRight`, `RightToLeft`, `Auto` |
| `Level` | 임베딩 레벨 (짝수 = LTR, 홀수 = RTL) |

## 구현한 부분과 미룬 부분 {#implemented-vs-deferred}

**구현됨** — 알고리즘은 규칙 **L2**까지 실행되며, 이는 유니코드 적합성 파일 `BidiCharacterTest.txt`가 다루는 전체 범위입니다.

- **P2, P3** 첫 번째 강한 문자로 결정하는 기본 문단 레벨.
- **X1–X8** 명시적 임베딩 *및* 아이솔레이트(오버플로 처리와 방향 상태 스택 포함), **X9** 폐기된 서식 문자와 `BN`의 제거, **X10** `sos`/`eos`를 갖는 아이솔레이팅 런 시퀀스.
- **W1–W7** 약한 유형.
- **N0** 짝 괄호 결정(BD16, U+2329/U+232A 정규 동등 포함), **N1–N2** 중립 유형.
- **I1, I2** 암시적 레벨.
- **L1** 구분자 / 후행 공백 재설정, **L2** 재배열.

**미룸** (bidi 엔진의 범위 밖이며 셰이퍼가 맡는 일):

- **L3**(결합 마크)과 **L4**(짝 괄호와 기타 미러링 문자의 글리프 미러링) — `VisualOrder`는 미러링된 글리프로 대체하지 않으며, 이는 렌더링/셰이핑 단계에 속합니다.
- 아랍어 연결 **셰이핑 / 조인** — [shape]({{< relref "shape.md" >}})를 참고하십시오.
- **P1** 문단 분리 — 호출자가 담당하며, API는 문단 단위로 동작합니다(문단 중간의 `Paragraph_Separator`는 여전히 X8/L1에서 처리됩니다).

## 적합성 {#conformance}

이 패키지는 `BidiCharacterTest.txt` **전체**로 검증됩니다(문단 레벨, 문자별 레벨, 시각적 순서의 모든 케이스 통과). 대표적인 케이스를 추린 부분 집합이 `testdata` 아래에 포함되어 있으며 `TestConformance`가 실행합니다. CI는 구문 커버리지 **정확히 100%**, `go vet`, `gofmt`, 그리고 64비트 아키텍처 여섯 개와 `js/wasm`, `darwin/arm64`, `windows/amd64`에 대한 크로스 컴파일을 강제합니다.

## 테이블 다시 생성하기 {#regenerating-the-tables}

```sh
go run ./cmd/genbidi .
```

이 명령은 유니코드 문자 데이터베이스에서 최신 `DerivedBidiClass.txt`와 `BidiBrackets.txt`를 가져와 `bidiclass_table.go`와 `bidibrackets_table.go`를 다시 씁니다.

소스: [github.com/go-typeset/bidi](https://github.com/go-typeset/bidi) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-typeset/bidi)
