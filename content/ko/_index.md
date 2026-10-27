---
title: "개요"
weight: 10
---

# go-opentype 문서 {#go-opentype-documentation}

**외부 의존성이 전혀 없는 순수 Go 기반 TrueType/OpenType 텍스트 스택입니다.** 작은 모듈 네 개가 함께 글꼴 파일을 읽고, 유니코드 양방향 텍스트와 복합 문자 텍스트를 처리하고, 그 결과를 래스터화(rasterisation)합니다. 스택 어디에도 `golang.org/x/*` 패키지나 cgo는 없습니다.

{{< callout type="info" >}}
**서드파티 의존성 없음**

스택의 모든 모듈은 Go 표준 라이브러리(`image`, `encoding/binary`, `math`, `errors` 등)만 임포트합니다. 덕분에 C 툴체인이나 별도로 포함된 에셋 없이도 `GOOS=js GOARCH=wasm`을 포함한 어떤 Go 타깃으로든 스택 전체를 손쉽게 크로스 컴파일할 수 있습니다.
{{< /callout >}}

## 모듈 {#the-modules}

| Repo | Module path | What it is |
| --- | --- | --- |
| [`opentype`]({{< relref "opentype.md" >}}) | `github.com/go-opentype/opentype` | 엔진 — sfnt 파싱 + 안티앨리어싱 래스터화 |
| [`bidi`]({{< relref "bidi.md" >}}) | `github.com/go-opentype/bidi` | 유니코드 양방향 알고리즘 (UAX #9) |
| [`shape`]({{< relref "shape.md" >}}) | `github.com/go-opentype/shape` | 위 두 모듈을 조합한 HarfBuzz-lite 복합 텍스트 셰이퍼 |
| [`fonts`]({{< relref "fonts.md" >}}) | `github.com/go-opentype/fonts` | 가독성이 좋은 내장 글꼴 패밀리 46개, 패밀리별 `go:embed` |
| [`docs`](https://github.com/go-opentype/docs) | — | 이 문서 사이트 (MkDocs Material, mike로 버전 관리) |
| [`brand`](https://github.com/go-opentype/brand) | — | 로고와 브랜드 에셋 |

각 모듈은 독립적이며 따로 임포트할 수 있습니다. `shape`만 나머지 두 모듈에 의존하고, `opentype`, `bidi`, `fonts`는 같은 조직 안의 다른 모듈에 의존하지 않습니다(`fonts`는 `Parse` 편의 래퍼 때문에 `opentype`의 `*Font` 타입에만 의존합니다).

## 무엇을 대체하는가 {#what-it-replaces}

`go-opentype`은 글리프(glyph)를 픽셀 버퍼에 그리는 Go 프로그램을 위해 다음을 기능적으로 대체하기 위해 존재합니다.

- 글리프를 그리는 UI에 필요한 `golang.org/x/image/font`와 `font/opentype`의 좁은 부분(글꼴 파싱, 크기가 지정된 페이스 생성, 어드밴스와 커버리지 마스크 가져오기), 그리고
- 왼쪽에서 오른쪽, 오른쪽에서 왼쪽 텍스트가 섞인 문장을 배치하기 위한 `golang.org/x/text/unicode/bidi`.

이 스택은 HarfBuzz + FreeType 조합과 기능 동등성을 목표로 하며, 다루는 범위에서는 이미 달성했습니다. TrueType `glyf` **및** CFF/CFF2 아웃라인 파싱과 래스터화, 모든 GSUB/GPOS 룩업 유형과 GDEF 마크 필터링, OpenType Variations(`fvar`/`avar`/`gvar`/`HVAR`/`VVAR`/`MVAR`), TrueType 인스트럭션 힌터와 CFF/Type 2 스템 그리드 피터, 세로쓰기 메트릭, 수식 조판 메트릭을 위한 OpenType `MATH` 테이블(HarfBuzz의 `hb-ot-math`를 따름), 규칙 L2까지의 완전한 UAX #9 양방향 알고리즘, 그리고 아랍어, 인도 계열 문자, 유니버설 셰이핑 엔진(Universal Shaping Engine, 태국어, 라오어, 크메르어, 미얀마어, 티베트어 등), 이집트 상형문자 쿼드랫, 한글, 세로쓰기(CJK 다테가키) 레이아웃, 라틴/기본 문자의 복합 텍스트 셰이핑을 지원합니다. 정확한 지원 범위는 각 모듈의 페이지를 참고하십시오.

## 설치 {#install}

```sh
go get github.com/go-opentype/opentype
go get github.com/go-opentype/bidi
go get github.com/go-opentype/shape
go get github.com/go-opentype/fonts
```

## 다음에 볼 내용 {#where-to-go-next}

- [opentype]({{< relref "opentype.md" >}}) — sfnt 파서와 래스터라이저: 글꼴, `cmap`, TrueType `glyf` 및 CFF/CFF2 아웃라인, 가변 글꼴, GSUB/GPOS 셰이핑, 힌팅(hinting), 세로쓰기 메트릭, 수식 조판을 위한 `MATH` 테이블, 글꼴 서브셋과 디스크립터 접근자, 그리고 전체 지원 매트릭스.
- [bidi]({{< relref "bidi.md" >}}) — UAX #9 엔진과, 구현한 부분 및 셰이퍼에 맡기는 부분.
- [shape]({{< relref "shape.md" >}}) — 복합 텍스트 셰이퍼: 아랍어 연결 조인, 인도 계열 문자 재배열, 유니버설 셰이핑 엔진, 이집트 상형문자 쿼드랫, 한글, 세로쓰기 레이아웃, 합자(ligature), 마크 부착, 커닝(kerning), 그리고 지원하는 문자 체계.
- [fonts]({{< relref "fonts.md" >}}) — 내장 패밀리 46개와 이를 사용하기 위한 패밀리별 임포트 모델.
- **가이드** — 작업 중심의 안내서: 텍스트를 PNG로 렌더링하기, 방향이 섞인 아랍어 문자열 셰이핑하기, 가독성을 위해 내장 글꼴 고르기.

소스는 [github.com/go-opentype](https://github.com/go-opentype)에 있습니다.
