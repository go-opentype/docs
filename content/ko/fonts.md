---
title: "fonts"
weight: 50
---

# fonts — 가독성 좋은 내장 패밀리 {#fonts-bundled-legible-families}

`github.com/go-opentype/fonts`는 가독성이 좋고 허용적 라이선스인 TrueType 글꼴 46개를 제공합니다. 패밀리마다 Go 하위 패키지가 하나씩 있고 각자 `//go:embed`를 가지므로, 내려받을 필요도 `.ttf`를 직접 구할 필요도 없으며 **바이너리에는 임포트한 패밀리만 링크됩니다**. 지원 범위는 라틴, 키릴, 그리스 문자와 비라틴 문자 일곱 가지(아랍어와 히브리어(RTL), 데바나가리, 태국어, 조지아어, 아르메니아어, 이집트 상형문자), 그리고 CJK(한자, 가나, 한글, 일반적인 CJK 문장 부호)를 위한 Noto Sans SC, JP, KR입니다. 순수 Go, 표준 라이브러리 전용 TrueType 엔진인 [opentype]({{< relref "opentype.md" >}})을 위해 만들었지만, `.ttf`를 받는 어떤 파서에서도 원시 바이트를 사용할 수 있습니다.

```go
import (
	"github.com/go-opentype/fonts"
	"github.com/go-opentype/fonts/inter"
)

f, err := fonts.Parse(inter.TTF) // *opentype.Font
face := f.NewFace(16)            // 16px face
```

순수 Go이며 cgo가 없고, 빌드 시점이나 실행 시점에 가져와야 할 외부 에셋이 없습니다. 모든 패밀리가 각자의 하위 패키지에 `//go:embed`로 포함되어 있습니다.

## 컴파일 시점에 지연되는 임포트 모델 {#the-lazy-at-compile-time-import-model}

`//go:embed`는 *패키지 단위로* 즉시 적용됩니다. 글꼴을 포함한 패키지는 그 글꼴을 쓰든 쓰지 않든, 해당 패키지를 임포트하는 모든 바이너리에 글꼴 바이트를 링크합니다. 46개 패밀리를 모두 한꺼번에 포함하는 `fonts` 패키지가 있다면, 무엇 때문이든 그 패키지를 임포트하는 순간 46개가 전부 바이너리에 들어갑니다.

그래서 루트 `fonts` 패키지는 그렇게 하지 않습니다. 각 패밀리는 자신의 `//go:embed`를 가진 별도의 하위 패키지(`fonts/inter`, `fonts/roboto`, `fonts/jetbrainsmono` 등)에 있습니다. `fonts/inter`를 임포트하면 Inter만 링크되고, 하위 패키지 열 개를 임포트하면 그 열 개만 링크됩니다.

루트 `fonts` 패키지에는 두 가지만 있습니다.

1. **[`Family`](#api) 메타데이터** — 내장된 모든 글꼴의 이름, [`Kind`](#api), 라이선스, 임포트 경로이며 `All`과 `ByName`이 반환합니다. `[]byte` 필드가 없으므로 패밀리를 나열해도 어떤 것도 링크되지 않습니다.
2. **[`MostLegible`](#api)** — 루트 패키지에 직접 포함된 유일한 패밀리로, 추가 임포트 없이 쓸 수 있는 합리적인 기본값입니다. 다른 모든 패밀리는 자신의 하위 패키지를 임포트해야 합니다.

## 내장 글꼴 {#bundled-fonts}

여섯 개 패밀리는 직접 선별한 것(v0.1.0부터 포함)이고, 나머지 마흔 개는 [google/fonts](https://github.com/google/fonts)에서 가져왔습니다. 비라틴 패밀리 일곱 개(아랍어, 히브리어, 데바나가리, 태국어, 조지아어, 아르메니아어, 이집트 상형문자)와 Noto Sans SC, JP, KR(내장 CJK 패밀리)이 라틴/키릴/그리스 문자 너머의 문자 지원 범위를 채웁니다.

| Name | Kind | License | Import path |
| --- | --- | --- | --- |
| Atkinson Hyperlegible | Sans | OFL-1.1 | `github.com/go-opentype/fonts/atkinsonhyperlegible` |
| Inter | Sans | OFL-1.1 | `github.com/go-opentype/fonts/inter` |
| Go | Sans | BSD-3-Clause | `github.com/go-opentype/fonts/goregular` |
| Lora | Serif | OFL-1.1 | `github.com/go-opentype/fonts/lora` |
| Go Mono | Mono | BSD-3-Clause | `github.com/go-opentype/fonts/gomono` |
| JetBrains Mono | Mono | OFL-1.1 | `github.com/go-opentype/fonts/jetbrainsmono` |
| Noto Sans Arabic (RTL) | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansarabic` |
| Noto Sans Hebrew (RTL) | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosanshebrew` |
| Noto Sans Devanagari | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansdevanagari` |
| Noto Sans Thai | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansthai` |
| Noto Sans Georgian | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansgeorgian` |
| Noto Sans Armenian | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansarmenian` |
| Noto Sans Egyptian Hieroglyphs | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansegyptianhieroglyphs` |
| Noto Sans SC (CJK) | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosanssc` |
| Noto Sans JP (CJK) | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosansjp` |
| Noto Sans KR (CJK) | Sans | OFL-1.1 | `github.com/go-opentype/fonts/notosanskr` |
| … 라틴/키릴/그리스 패밀리 30개 더 (Roboto, Open Sans, Montserrat, Fira Sans/Code, IBM Plex 등) | Sans/Serif/Mono | OFL-1.1 | 전체 목록은 [pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts)를 참고하십시오 |

전체 라이선스 문서는 `licenses/` 아래에 원문 그대로 포함되어 있으며, 패밀리마다 파일이 하나씩 있습니다(`OFL-1.1` 행은 모두 SIL Open Font License 1.1이고, Go와 Go Mono가 함께 쓰는 `GoFonts-LICENSE.txt`는 BSD-3-Clause, Bigelow & Holmes입니다).

{{< callout type="info" >}}
**기본값: Atkinson Hyperlegible**

Braille Institute가 저시력 독자를 위해 문자 구분을 극대화하도록 설계했으며, 그 결과 모든 독자에게 도움이 됩니다. [`MostLegible()`](#api)이 이 글꼴을 반환하며, 루트 패키지에 직접 포함된 유일한 패밀리입니다.
{{< /callout >}}

{{< callout type="info" >}}
**내장 인스턴스는 정적이지만 엔진 자체는 가변 글꼴을 지원합니다**

**Inter**, **Lora**, **JetBrains Mono**와 `google/fonts`에서 가져온 대부분의 글꼴은 원본이 가변 글꼴이며, 번들을 단순하고 작게 유지하기 위해 내장된 각 `.ttf`는 해당 패밀리의 기본 마스터(정적 인스턴스)로 고정되어 있습니다. 이는 패키징상의 선택이지 엔진의 한계가 아닙니다. [opentype]({{< relref "opentype.md#variable-fonts" >}})은 `(*Face).SetVariation`을 통해 OpenType Variations를 완전히 지원합니다. 특정 축 좌표(예: 두께 600의 Inter)가 필요하다면 해당 패밀리의 가변 `.ttf`를 직접 구해 `opentype.Parse`로 파싱하십시오.
{{< /callout >}}

## API {#api}

```go
type Kind int

const (
	KindSans Kind = iota
	KindSerif
	KindMono
	KindDisplay
)

// Family is metadata only — no font bytes. Fetch bytes via ImportPath.
type Family struct {
	Name       string // e.g. "Inter"
	Kind       Kind
	License    string // SPDX id: "OFL-1.1" or "BSD-3-Clause"
	ImportPath string // e.g. "github.com/go-opentype/fonts/inter"
}

func All() []Family                        // every bundled family's metadata, stable order
func ByName(name string) (Family, bool)     // case-insensitive lookup by Family.Name
func MostLegible() []byte                   // Atkinson Hyperlegible bytes — the one family embedded here
func Parse(ttf []byte) (*opentype.Font, error) // convenience wrapper over opentype.Parse
```

그 밖의 모든 패밀리의 바이트는 각자의 하위 패키지에 `var TTF []byte`로 들어 있습니다(예: `inter.TTF`, `roboto.TTF`, `jetbrainsmono.TTF`). 임포트 경로 전체 목록은 위의 표와 [pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts)를 참고하십시오.

## 라이선스 {#license}

저장소의 Go 소스 코드(내장된 `.ttf` 파일과 `licenses/`를 제외한 모든 것)는 BSD-3-Clause입니다. 내장된 글꼴 파일은 각자의 원본 라이선스를 그대로 따릅니다. 전체 문서, 저작권 고지, 그리고 (OFL 글꼴의 경우) 예약 글꼴 이름(Reserved Font Names)은 위의 표와 `licenses/`를 참고하십시오.

소스: [github.com/go-opentype/fonts](https://github.com/go-opentype/fonts) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts)
