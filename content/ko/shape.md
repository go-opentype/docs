---
title: "shape"
weight: 40
---

# shape — 복합 텍스트 셰이퍼 {#shape-the-complex-text-shaper}

`github.com/go-opentype/shape`는 순수 Go, `CGO_ENABLED=0`, **표준 라이브러리 전용** 복합 텍스트 셰이퍼로, go-opentype 스택을 위한 HarfBuzz-lite입니다. 유니코드 텍스트 한 구간을 **시각적 순서**로 배치된 글리프(glyph)로 변환하므로, 아랍어, 인도 계열 문자, 동남아시아 문자, CJK 세로쓰기, 이집트 상형문자 텍스트가 서로 떨어지고 연결되지 않고 재배열되지 않은 글리프가 아니라 올바르게 렌더링됩니다.

이 모듈은 두 형제 모듈을 조합할 뿐 그 외에는 아무것도 추가하지 않습니다. [`bidi`]({{< relref "bidi.md" >}})의 양방향 재배열과 [`opentype`]({{< relref "opentype.md" >}})의 GSUB/GPOS 기반 치환/배치를 사용합니다. `golang.org/x/*`도 서드파티 모듈도 없으며, `GOOS=js GOARCH=wasm`을 포함한 모든 Go 타깃에서 빌드됩니다.

## 하는 일 {#what-it-does}

- **양방향 재배열** — UAX #9 임베딩 레벨을([`bidi`]({{< relref "bidi.md" >}})를 통해) 결정하고 글리프를 왼쪽에서 오른쪽으로 배치하므로, 오른쪽에서 왼쪽으로 쓰는 구간이 그리는 순서대로 출력됩니다.
- **아랍어 연결 조인** — 각 글자의 조인 형태(독립형 / 어두형 / 어중형 / 어말형)를 결정하고, 글꼴의 `isol`/`init`/`medi`/`fina` GSUB 피처를 해당 형태의 글리프에만 **위치별로** 적용합니다. 조인 형태는 `ccmp` 분해 과정에서도 추적되므로(Noto Sans Arabic 같은 실제 글꼴이 쓰는 rasm 골격 + 점 구조), 조인된 글리프가 실제로 이어집니다.
- **인도 계열 셰이핑** — 데바나가리, 벵골어, 구르무키, 구자라트어, 오리야어, 타밀어, 텔루구어, 칸나다어, 말라얄람어, 싱할라어를 HarfBuzz의 "indic" 모델로 처리합니다. 음절 분리, 기저/reph 탐지, 두 차례의 재배열(기저 앞 마트라와 reph), 완전한 기본 + 표현 GSUB 피처 파이프라인, GPOS mark/mkmk/abvm/blwm 부착을 포함합니다.
- **유니버설 셰이핑 엔진 (Universal Shaping Engine, USE)** — 전용 셰이퍼가 없는 문자 체계를 위한 범용 복합 문자 모델로, 태국어, 라오어, 크메르어, 미얀마어, 티베트어, 자바어, 발리어, 부기스어, 타이 탐 등을 다룹니다. 런을 USE 음절 범주로 분류하고, 클러스터로 나누고, 재배열(기저 앞 모음과 수식 기호는 기저 앞으로, repha는 기저 뒤로)한 다음 USE GSUB/GPOS 파이프라인에 통과시키며, sakot/halant 조인, 분리 모음 분해, 불완전한 클러스터에 대한 점선 원 삽입을 포함합니다.
- **이집트 상형문자 쿼드랫** — 유니코드 서식 제어 문자 U+13430–U+1345F(조이너, 모서리 삽입, 오버레이, 세그먼트/인클로저 구분자)를 2차원 쿼드랫 트리로 파싱해 기하학적으로 배치하므로, 기호의 나열이 조밀한 블록으로 렌더링됩니다.
- **한글** — 자모를 미리 조합된 음절 블록으로 조합합니다.
- **세로쓰기 모드 (CJK 다테가키)** — `Options.Vertical`은 `vert`/`vrt2` 직립 글리프 형태를 선택하고, 글꼴의 세로 메트릭(`vmtx`/`VORG`)을 사용해 글리프를 위에서 아래로 쌓습니다.
- **합자(ligature), 마크 부착, 커닝(kerning)** — 모든 문자 체계 경로에서 GSUB `ccmp`/`rlig`/`liga`/`calt`에 이어 GPOS `kern`/`mark`/`mkmk`/`curs`를 적용하므로, 발음 구별 기호가 기저 문자 위에 놓이고 글자 쌍이 커닝됩니다.

## 사용법 {#usage}

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

기본 방향은 `Auto`(첫 번째 강한 문자에서 결정)이고, 문자 체계는 텍스트에서 자동으로 감지됩니다(아랍어 블록의 룬이 하나라도 있으면 아랍어를, 인도 계열/USE 룬은 해당 셰이퍼를 선택하는 식). `Options.Script`나 `Options.Direction`을 지정하면 이 동작을 바꿀 수 있습니다.

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

## 범위 {#scope}

**구현됨:** 아랍어, 전용 셰이퍼가 있는 **인도 계열** 문자 열 가지, **유니버설 셰이핑 엔진**(태국어, 라오어, 크메르어, 미얀마어, 티베트어, 자바어, 발리어, 부기스어, 타이 탐 등), **이집트 상형문자** 쿼드랫 레이아웃, **한글** 자모 조합, **세로쓰기** 모드, 그리고 **라틴/기본**(라틴, 키릴, 그리스, 가로쓰기 CJK 등) 셰이핑.

클러스터 인덱스는 일대일 치환(아랍어 위치별 형태)에서는 정확하며, 치환으로 런의 길이가 바뀌는 경우(합자, 분해, 인도 계열/USE 재배열)에는 최선의 노력으로 단조성을 유지하는 값입니다.

소스: [github.com/go-opentype/shape](https://github.com/go-opentype/shape) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/shape)
