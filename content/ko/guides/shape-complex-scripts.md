---
title: "복합 문자 셰이핑하기"
weight: 80
---

# 가이드: 복합 문자 셰이핑하기 {#guide-shape-complex-scripts}

글리프(glyph)마다 어드밴스를 더하는 단순한 룬별 루프([텍스트를 PNG로 렌더링하기]({{< relref "render-text-to-png.md" >}})와 같은 방식)는 고립된 라틴 텍스트에는 충분하지만, **재배열**이나 **연결 조인**이 필요한 경우, 즉 오른쪽에서 왼쪽으로 쓰는 아랍어 구간이나 이웃 글자와 이어져야 하는 아랍어 글자에는 통하지 않습니다. 이럴 때 쓰는 것이 [`shape`]({{< relref "../shape.md" >}})입니다.

## 오른쪽에서 왼쪽으로 쓰는 아랍어 문자열 {#a-right-to-left-arabic-string}

```go
package main

import (
	"fmt"

	"github.com/go-opentype/opentype"
	"github.com/go-opentype/shape"
)

func main() {
	f, err := opentype.Parse(arabicTTF) // a font with Arabic coverage, e.g. Noto Sans Arabic
	if err != nil {
		panic(err)
	}
	face := f.NewFace(32)

	glyphs := shape.Shape(face, "بيت", shape.Options{}) // "house"

	penX, penY := 0, 0
	for _, g := range glyphs {
		x := penX + g.XOffset
		y := penY + g.YOffset
		fmt.Printf("draw glyph %d at (%d,%d)\n", g.GID, x, y)
		penX += g.XAdvance
		penY += g.YAdvance
	}
}
```

`shape.Shape`는 원본 문자열이 논리적으로는 오른쪽에서 왼쪽이더라도 글리프를 이미 **시각적 순서**, 즉 그리는 순서대로 왼쪽에서 오른쪽으로 반환합니다. 이를 위해 [`bidi`]({{< relref "../bidi.md" >}})를 직접 호출할 필요는 없습니다. `shape`가 내부에서 조합해 사용합니다.

## 방향이 섞인 텍스트 {#mixed-direction-text}

문자 체계가 섞인 문단(아랍어 안의 영어, 또는 그 반대)에서는 `Options.Direction`을 기본값인 `Auto`로 두십시오. 기본 방향은 UAX #9의 규칙 P2/P3에 따라 첫 번째 강한 문자에서 정해지고, 문자열 안의 방향별 런은 각각 독립적으로 셰이핑되고 재배열된 뒤 시각적 순서로 이어 붙여집니다.

```go
glyphs := shape.Shape(face, "The word بيت means house", shape.Options{})
```

문단(또는 임베딩) 방향을 자동 감지에 맡기지 않고 고정해야 한다면, 예를 들어 첫 글자와 관계없이 항상 RTL로 배치하는 UI라면 명시적으로 지정하십시오.

```go
glyphs := shape.Shape(face, text, shape.Options{
    Direction: bidi.RightToLeft,
})
```

## 문자 체계 강제 지정 {#forcing-a-script}

문자 체계는 자동으로 감지됩니다(아랍어 블록의 룬이 하나라도 있으면 아랍어 셰이퍼가 선택되고, 그 외에는 라틴/기본 경로를 탑니다). 자동 감지보다 사용자가 더 잘 아는 경우에는 `Options.Script`로 덮어쓰십시오. 예를 들어 라틴 문자만 쓰는 UI 문자열에 우연히 아랍어 문장 부호가 들어 있어도 연결 조인이 일어나지 않게 하려는 경우입니다.

```go
glyphs := shape.Shape(face, text, shape.Options{Script: "latn"})
```

## 인도 계열 문자와 유니버설 셰이핑 엔진 문자 {#indic-and-universal-shaping-engine-scripts}

데바나가리, 벵골어, 타밀어를 비롯해 전용 셰이퍼가 있는 인도 계열 문자 열 가지는 HarfBuzz의 "indic" 모델 전체를 적용받습니다. 음절 분리, 기저/reph 탐지, 기저 앞 마트라와 reph 재배열 단계, 그리고 (mark/mkmk/abvm/blwm 부착을 포함한) 계층적 GSUB/GPOS 피처 파이프라인입니다.

```go
glyphs := shape.Shape(face, "नमस्ते", shape.Options{}) // Devanagari "hello"
```

태국어, 라오어, 크메르어, 미얀마어, 티베트어를 비롯해 전용 셰이퍼가 없는 문자 체계는 유니버설 셰이핑 엔진(Universal Shaping Engine, USE)을 거칩니다. USE는 런을 음절 범주로 분류하고, 클러스터로 나누고, 기저 앞 모음/수식 기호와 repha를 재배열하며, USE GSUB/GPOS 파이프라인(sakot/halant 조인, 분리 모음 분해, 불완전한 클러스터에 대한 점선 원 삽입 포함)을 실행합니다. 문자 체계 감지가 자동이므로 `shape.Shape`를 호출하는 것 외에 코드를 바꿀 필요가 없습니다. 각 모델이 다루는 문자 체계의 전체 목록은 [shape의 범위]({{< relref "../shape.md#scope" >}})를 참고하십시오.

## 세로쓰기(CJK 다테가키)와 기타 문자 체계 {#vertical-cjk-tategaki-and-other-scripts}

이집트 상형문자 쿼드랫과 한글 자모 조합도 입력 텍스트의 문자 체계에 따라 자동으로 처리됩니다. 세로쓰기 모드(CJK 다테가키 — 위에서 아래로, `vert`/`vrt2` 글리프 형태)에는 `Options.Vertical`을 설정하십시오.

```go
glyphs := shape.Shape(face, "こんにちは", shape.Options{Vertical: true})
```

`shape.Shape`는 아랍어, 인도 계열 문자, USE, 이집트 상형문자, 한글, 세로쓰기, 그리고 라틴/기본(키릴, 그리스, 가로쓰기 CJK도 포함)까지 이 모든 경로를 아우르는 완전한 해법입니다.
