---
title: "접근성 좋은 글꼴 고르기"
weight: 90
---

# 가이드: 접근성 좋은 글꼴 고르기 {#guide-choose-an-accessible-font}

[`fonts`]({{< relref "../fonts.md" >}})는 Go 프로그램이 `.ttf`를 구하거나 내려받지 않고도 가독성 좋은 텍스트를 제공할 수 있도록 패밀리 46개를 내장합니다. 이 가이드는 기본값만이 아니라 알맞은 글꼴을 고르는 방법을 다룹니다.

## 그냥 읽기 편한 글꼴이면 충분하다면? 기본값을 쓰십시오 {#just-want-something-legible-use-the-default}

```go
f, err := fonts.Parse(fonts.MostLegible())
```

`MostLegible()`은 **Atkinson Hyperlegible**을 반환합니다. 미국 Braille Institute가 저시력 독자를 위해 문자 구분을 극대화하도록 설계한 글꼴로, 대부분의 산세리프에서 가장 헷갈리기 쉬운 글자(`I`/`l`/`1`, `O`/`0`)를 의도적으로 구별되게 만들었습니다. 저시력 독자만이 아니라 모든 독자에게 도움이 되므로, 선택 사항이 아니라 패키지의 기본값으로 삼았습니다.

## 역할별로 고르기 {#picking-by-role}

`fonts.All()`은 내장된 모든 `Family`를 반환하며, 각각에 `Kind`(`KindSans`, `KindSerif`, `KindMono`)가 붙어 있으므로 이름을 하드코딩하는 대신 역할로 거를 수 있습니다.

```go
for _, fam := range fonts.All() {
    if fam.Kind == fonts.KindMono {
        // e.g. Go Mono or JetBrains Mono — a code block face
    }
}
```

또는 이름으로 찾아 임포트 경로를 알아낼 수도 있습니다(`Family`에는 `[]byte`가 없으므로 패밀리를 나열하거나 검색해도 바이트가 링크되지 않습니다. 바이트 자체를 쓰려면 여전히 하위 패키지를 임포트해야 합니다).

```go
fam, ok := fonts.ByName("Lora") // case-insensitive
if ok {
    fmt.Println(fam.ImportPath) // github.com/go-opentype/fonts/lora
}
// import "github.com/go-opentype/fonts/lora", then:
f, err := fonts.Parse(lora.TTF)
```

## 무엇이 내장되어 있고 왜 고르는가 {#whats-bundled-and-why-youd-pick-it}

| Family | Kind | Best for |
| --- | --- | --- |
| **Atkinson Hyperlegible** | Sans | 기본 선택. 가독성이 가장 중요한 본문 텍스트 |
| Inter | Sans | UI 크롬, 밀도 높은 인터페이스, 더 중립적인 기하학적 산세리프 |
| Go | Sans | Go 프로젝트 자체의 산세리프 — Go 도구용 UI에 자연스러운 선택 |
| Lora | Serif | 긴 글 읽기, 편집 디자인 맥락 |
| Go Mono | Mono | 코드. 위의 Go 산세리프와 어울림 |
| JetBrains Mono | Mono | 코드. 합자(ligature)에 적합한 더 넓은 고정폭 글꼴 |

## 가변 글꼴에 관한 주의점 {#the-variable-font-caveat}

Inter, Lora, JetBrains Mono는 원본이 **가변 글꼴**(두께/너비 축 범위 전체를 하나의 파일에 담은 것)입니다. `go-opentype/fonts`는 번들을 단순하고 작게 유지하기 위해, 엔진이 축을 다루지 못해서가 아니라, 각각을 **기본 정적 인스턴스**로 고정합니다. 여기에 내장된 `.ttf`는 가변 범위 전체가 아니라 고정된 하나의 두께입니다. [`opentype`은 OpenType Variations를 완전히 지원하며]({{< relref "../opentype.md#variable-fonts" >}}), 어떤 축 좌표든 인스턴스화하는 `SetVariation`도 포함합니다. 디자인에 축의 특정 두께가 필요하다면(예: 기본값 400이 아닌 두께 600의 Inter), 예를 들어 [google/fonts](https://github.com/google/fonts)에서 해당 패밀리의 가변 `.ttf`를 직접 구해 `opentype.Parse` + `(*Face).SetVariation(map[string]float64{"wght": 600})`로 사용하십시오.

## 라이선스 의무 {#license-obligations}

`fonts`의 Go 소스는 BSD-3-Clause이지만, **글꼴 파일 자체**는 원본 라이선스를 그대로 따릅니다. Atkinson Hyperlegible, Inter, Lora, JetBrains Mono는 OFL-1.1이고, Go/Go Mono는 BSD-3-Clause입니다. 이 글꼴을 내장한 바이너리를 재배포한다면 `licenses/`의 해당 라이선스 문서를 함께 제공하십시오. 전체 표와 저작권 표기는 [fonts]({{< relref "../fonts.md#bundled-fonts" >}})를 참고하십시오.
