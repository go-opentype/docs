---
title: "opentype"
weight: 20
---

# opentype — 엔진 {#opentype-the-engine}

`github.com/go-opentype/opentype`은 TrueType/OpenType 글꼴을 위한 순수 Go, `CGO_ENABLED=0`, **표준 라이브러리 전용** 파서이자 안티앨리어싱 래스터라이저입니다. `image`, `encoding/binary`, `math`, `errors` 등만 임포트하며 `golang.org/x/*`나 서드파티 모듈이 없으므로, `GOOS=js GOARCH=wasm`을 포함한 모든 Go 타깃에서 빌드됩니다.

글리프(glyph)를 그리는 UI에 필요한 `golang.org/x/image/font/opentype`의 좁은 부분, 즉 글꼴 블롭 파싱, 픽셀 크기의 페이스 생성, 룬별 어드밴스와 8비트 알파 커버리지 마스크 가져오기를 기능적으로 대체하기 위해 만들어졌으며, 그 너머로도 나아갑니다. CFF/CFF2 아웃라인, 가변 글꼴, GPOS/GSUB 셰이핑, 커닝(kerning), TrueType/CFF 힌팅(hinting)이 모두 범위에 들어 있습니다.

## 사용법 {#usage}

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

`GlyphMask`의 형태는 `x/image`의 `font.Face.Glyph`를 의도적으로 따랐으므로, x/image 페이스를 이것으로 바꾸는 작업은 기계적으로 할 수 있습니다.

## 가변 글꼴 {#variable-fonts}

`SetVariation`으로 가변 글꼴을 디자인 공간의 특정 지점에서 인스턴스화합니다. 축 태그(`wght`, `wdth`, `opsz` 등)를 키로 사용합니다.

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

`SetVariation`은 글리프 아웃라인을 다시 보간하고(TrueType는 `gvar`, CFF2는 CFF2 블렌드), 선택한 좌표에 맞춰 `HVAR`/`VVAR`/`MVAR` 기반 메트릭을 다시 계산합니다. `avar` 축 값 매핑은 자동으로 적용됩니다.

## 힌팅 {#hinting}

`SetHinting(true)`는 래스터화 전에 TrueType 인스트럭션 인터프리터(`glyf` 아웃라인용) 또는 CFF/Type 2 스템 그리드 피터(CFF/CFF2 아웃라인용)를 실행하고, `SetStemDarkening(true)`는 여기에 더해 CFF 스템 다크닝을 적용합니다. 둘 다 기본값은 꺼짐이며(균일한 4×4 슈퍼샘플링 렌더링).

## 수식 조판 (`MATH` 테이블) {#mathematical-typesetting-the-math-table}

`opentype`은 OpenType **`MATH`** 테이블을 디코딩합니다. 수식 조판 엔진(LuaTeX, unicode-math, MathJax, Word의 수식 편집기, 또는 HarfBuzz의 `hb-ot-math` 기반의 모든 것)이 수식을 배치하는 데 필요한 글꼴 수준의 메트릭입니다. 세 부분으로 이루어지며, 모두 `Face`를 통해 픽셀 단위로 환산되어 노출됩니다.

- **MathConstants** — 약 56개의 전역 스칼라 값: 수식 축 높이, 아래첨자/위첨자 이동량과 간격 최솟값, 분수와 근호의 간격, 선 두께 등이며, `MathConstant`와 `opentype.AxisHeight` 또는 `opentype.FractionRuleThickness` 같은 `MathConstant` 값으로 읽습니다(디스크상 순서의 전체 목록은 `go doc`을 참고하십시오).
- **MathGlyphInfo** — 글리프별 *이탤릭 보정*(`ItalicCorrection`, 기울어진 글리프 뒤에 직립 글리프가 올 때 넣을 간격, 또는 위첨자 뒤에 이어지는 아래첨자의 가로 오프셋), *상단 악센트 부착*(`TopAccentAttachment`, 글리프 위에 악센트를 가운데 맞출 x 위치), *확장 형태* 플래그(`IsExtendedShapeGlyph`, 큰 적분 기호처럼 글리프가 늘어난 것처럼 위첨자가 배치되는 글리프용), 그리고 네 모서리의 컷인 *수식 커닝*(`MathKern`, 인접한 첨자의 부착 높이에 대한 계단 함수).
- **MathVariants** — 글리프의 더 큰 크기 변형과, 가장 큰 변형으로도 충분하지 않을 때 반복 가능한 부품으로 임의의 높이나 너비의 늘어나는 글리프(적분 기호, 중괄호, 근호, 화살표)를 조립하는 방법이며, 둘 다 `MathVariants`가 함께 반환합니다.

수식 *레이아웃* 자체, 즉 박스 모델, TeX의 수식 리스트 규칙, 변형 선택이나 목표 크기에 맞춘 조립 확장은 이 메트릭 위에 구축되는 상위 수준의 문제이며, `hb-ot-math`가 메트릭만 노출하고 레이아웃은 호출자에게 맡기는 것과 똑같이 이 패키지의 범위 밖입니다.

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

실제 STIX Two Math 글꼴을 em당 32px로 실행하면 `axis height: 8 fraction rule thickness: 2`, `italic correction: 1`, `size variants: 13 assembly parts: 3`이 출력됩니다. STIX Two Math(SIL Open Font License)는 `opentype` 저장소에 [`testdata/STIXTwoMath-Regular.otf`](https://github.com/go-opentype/opentype/blob/main/testdata/STIXTwoMath-Regular.otf)로 포함되어 있으며, 디코더에 100% 분기 커버리지를 제공하는 합성 MATH 테이블 픽스처와 함께 CI에서 처음부터 끝까지 검증됩니다.

## 글꼴 디스크립터와 서브셋 {#font-descriptors-and-subsetting}

글꼴을 *내장하는* 프로그램, 예를 들어 PDF 작성기나 CSS `@font-face` 생성기를 위해 `opentype`은 쓰기 쪽 기본 요소도 노출합니다.

- **디스크립터 스칼라** — OS/2와 `head`/`post`에서 가져온 값으로, PDF `/FontDescriptor`나 CSS 선언을 채우는 데 바로 쓸 수 있습니다: `Ascent`, `Descent`, `LineGap`, `FontBBox`, `CapHeight`, `XHeight`, `ItalicAngle`, `WeightClass`, `WidthClass`, `StemV`, `Flags`, 그리고 `IsFixedPitch`/`IsItalic`/`IsSerif` 불리언.
- **인스턴싱** — `Instance(coords)`는 가변 글꼴의 디자인 공간 한 지점으로 구워진 새 `*Font`(glyf/CFF2 아웃라인과 메트릭이 고정됨)를 반환하고, `InstanceBytes(coords)`는 그 인스턴스를 새 sfnt 블롭으로 반환합니다.
- **서브셋** — `SubsetTrueType(gids)`는 TrueType 글꼴을 요청한 글리프로만 줄이며, 복합 글리프의 의존성을 닫고 이전→새 글리프 ID 매핑을 반환합니다. `SubsetCFF(gids)`는 CFF/OTTO 글꼴에 같은 일을 하며 `seac` 악센트 참조를 닫습니다. 둘 다 문서가 실제로 쓰는 글리프만 남기므로 내장 글꼴이 작게 유지됩니다.

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

## WOFF2와 바이너리 크기 {#woff2-and-binary-size}

**상태: main에 있으며 다음 릴리스에 포함됩니다** (병합되었고 아직 태그는 없습니다).

WOFF2 글꼴은 이제 `github.com/andybalholm/brotli`가 아니라 디코더 전용인 `github.com/dsnet/compress/brotli`로 압축을 풉니다. `andybalholm/brotli`의 루트 패키지는 `net/http`를 임포트하기 때문에, `opentype`을 임포트하는 모든 프로그램에 `net/http`와 `crypto/tls`의 `init`이 링크되었습니다. 7.5 MB의 `GOOS=js` 텍스트 프로그램 중 1.8 MB가 여기에 해당했습니다. 디코더 전용 패키지는 둘 다 임포트하지 않습니다.

압축 해제는 이제 WOFF2 디렉터리가 선언한 크기(최대 256 MiB)로 제한되므로, 압축 폭탄은 할당되지 않고 거부됩니다.

미해결 항목(수정이 아닌 트레이드오프): `dsnet/compress/brotli`는 압축 해제가 약 1.37배 느리며, 200 KB 글꼴에서 대략 0.4 ms입니다. 크기와 제한을 얻기 위해 받아들인 것이며, 나중에 다시 검토할 수 있도록 여기에 기록해 둡니다.

## API {#api}

| Symbol | Purpose |
| --- | --- |
| `Parse(b []byte) (*Font, error)` | sfnt(TrueType 또는 CFF/OTTO) 글꼴 블롭을 디코딩합니다. |
| `(*Font).NumGlyphs() int` | 글리프 수(`maxp`에서). |
| `(*Font).GlyphIndex(r rune) (GlyphIndex, bool)` | cmap으로 룬을 매핑합니다. |
| `(*Font).GlyphIndexVariation(r, vs rune) (GlyphIndex, bool)` | 룬 + 유니코드 이체자 선택자를 매핑합니다. |
| `(*Font).Axes() []Axis` | 가변 글꼴의 디자인 축(`fvar`). |
| `(*Font).NamedInstances() []NamedInstance` | 변형 공간의 이름 붙은 위치. |
| `(*Font).GPOS() *GPOS` / `(*Font).GSUB() *GSUB` | 파싱된 레이아웃 테이블, 없으면 `nil`. |
| `(*Font).HasVerticalMetrics() bool` | `vhea`/`vmtx`가 있는지 여부. |
| `(*Font).NewFace(sizePx int) *Face` | 크기가 지정된 페이스를 만듭니다(스케일 = `sizePx / unitsPerEm`). |
| `(*Face).Metrics() Metrics` | 픽셀 단위의 `Ascent`, `Descent`, `Height`. |
| `(*Face).Advance(r rune) int` / `(*Face).Measure(s string) int` | 룬 / 문자열의 픽셀 단위 어드밴스. |
| `(*Face).Kern(prev, r rune) int` / `(*Face).MeasureKerned(s string) int` | GPOS 또는 `kern`의 글자 쌍 조정. |
| `(*Face).GlyphMask(r, x, y) (image.Rectangle, *image.Alpha, image.Point, int, bool)` | 래스터화된 글리프 + 어드밴스. |
| `(*Face).Shape(text string, features ...string) []GlyphIndex` | GSUB로 치환된 글리프 런. |
| `(*Face).ShapePositioned(text string, features ...string) []PositionedGlyph` | 펜 오프셋을 포함한 GSUB + GPOS 글리프 런. |
| `(*Face).SetVariation(coords map[string]float64)` | 주어진 축 좌표에서 가변 글꼴을 인스턴스화합니다. |
| `(*Face).SetHinting(on bool)` / `(*Face).SetStemDarkening(on bool)` | TrueType/CFF 힌터를 켜고 끕니다. |
| `(*Face).VerticalAdvance(r rune) int` / `(*Face).VerticalOrigin(r rune) (int, bool)` | 세로쓰기 모드 메트릭. |
| `(*Font).HasMath() bool` | 글꼴에 OpenType `MATH` 테이블이 있는지 여부. |
| `(*Font).IsExtendedShapeGlyph(gid GlyphIndex) bool` | gid가 `MATH` 확장 형태 커버리지에 속하는지 여부. |
| `(*Face).MathConstant(which MathConstant) int` | 픽셀 단위로 환산된 약 56개의 `MATH` 레이아웃 상수 중 하나(축 높이, 첨자 이동, 분수/근호 간격 등). |
| `(*Face).ItalicCorrection(gid GlyphIndex) int` | 글리프별 수식 이탤릭 보정, 픽셀 단위. |
| `(*Face).TopAccentAttachment(gid GlyphIndex) (int, bool)` | gid 위에 상단 악센트를 가운데 맞출 X 위치(픽셀). |
| `(*Face).MathKern(gid GlyphIndex, corner MathKernCorner, correctionHeight int) int` | 주어진 부착 높이에서 gid의 네 모서리 중 하나에 대한 컷인 수식 커닝(픽셀). |
| `(*Face).MathVariants(gid GlyphIndex, vertical bool) ([]MathVariant, *MathAssembly)` | 한 축을 따른 gid의 크기 변형과 늘어나는 글리프 조립. |
| `(*Font).Ascent/Descent/LineGap/FontBBox/CapHeight/XHeight/ItalicAngle/WeightClass/WidthClass/StemV/Flags/IsFixedPitch/IsItalic/IsSerif` | PDF `/FontDescriptor`와 CSS `@font-face`를 위한, OS/2 + `head`/`post`의 글꼴 디스크립터 스칼라. |
| `(*Font).Instance(coords map[string]float64) (*Font, error)` / `(*Font).InstanceBytes(coords) ([]byte, error)` | 가변 글꼴을 정적 인스턴스(`*Font` 또는 새 sfnt 블롭)로 굽습니다. |
| `(*Font).SubsetTrueType(gids []GlyphIndex) ([]byte, map[GlyphIndex]GlyphIndex, error)` | TrueType 글꼴을 글리프 집합으로 서브셋하며 복합 글리프를 닫고, 이전→새 ID 맵을 반환합니다. |
| `(*Font).SubsetCFF(gids []GlyphIndex) ([]byte, error)` | CFF/OTTO 글꼴을 글리프 집합으로 서브셋하며 `seac` 악센트를 닫습니다. |

`Font`는 `Parse` 이후 불변이며 동시에 사용해도 안전합니다. `Face`는 래스터화된 글리프를 캐시하며 동시에 사용하기에 안전하지 **않습니다**. 필요하다면 고루틴마다 `Face`를 하나씩 만드십시오.

## 지원 매트릭스 {#support-matrix}

- sfnt 컨테이너(`0x00010000`과 `true` TrueType 매직, `OTTO` CFF 매직)
- `head`, `maxp`, `hhea`, `hmtx`(끝부분의 공유 어드밴스 런 포함)
- `cmap` 형식 **0, 2, 4, 6, 8, 10, 12, 13** 및 **14**(유니코드 이체자 시퀀스), 12를 우선 사용
- `loca`(짧은 형식과 긴 형식), `glyf` 단순 글리프(반복 플래그, 짧은/긴 델타)와 복합 글리프(`ARGS_ARE_XY_VALUES`, scale / x&y-scale / 2×2, `MORE_COMPONENTS`, 순환 및 깊이 가드 포함)
- 2차 곡선 윤곽에 대한 암시적 온커브 중간점 합성
- CFF와 CFF2(`OTTO`) Type 2 차스트링 아웃라인, 서브루틴, `seac` 악센트 합성, CFF2 가변 글꼴 블렌드 연산자 포함
- `fvar` 축과 이름 붙은 인스턴스, `avar` 축 값 매핑, `gvar`/CFF2 글리프 아웃라인 보간, `HVAR`/`VVAR`/`MVAR` 메트릭 변형, `(*Face).SetVariation`을 통해 제공
- 모든 `GSUB`/`GPOS` 룩업 유형 — 합자(ligature), 문맥 및 위치별 형태 치환(`isol`/`init`/`medi`/`fina`), 단일/쌍/연결 위치 지정, mark-to-base와 mark-to-mark 부착 — 그리고 `GDEF`와 룩업 플래그 마크 필터링
- GPOS 쌍 위치 지정과 기존 `kern` 테이블 폴백을 통한 커닝(kerning), 단일 `Kerner`로 제공
- TrueType 인스트럭션 힌터와 CFF/Type 2 스템 그리드 피터(블루 존, 카운터, 스템 다크닝), `Face`별로 `SetHinting` / `SetStemDarkening`으로 전환
- 세로쓰기 모드를 위한 `vhea`/`vmtx`/`VORG` 기반 세로 메트릭과 원점
- OpenType **`MATH`** 테이블 — 수식 조판 메트릭(상수, 글리프별 이탤릭 보정 / 상단 악센트 부착 / 네 모서리 커닝, 늘어나는 글리프의 크기 변형과 조립)을 `Face`(`HasMath`, `MathConstant`, `ItalicCorrection`, `TopAccentAttachment`, `MathKern`, `MathVariants`, `IsExtendedShapeGlyph`)를 통해 픽셀 단위로 환산해 노출. 수식 *레이아웃*은 상위 수준 엔진에 맡김
- PDF와 CSS 내장을 위한 OS/2 + `head`/`post` **글꼴 디스크립터** 스칼라(`Ascent`, `Descent`, `FontBBox`, `CapHeight`, `XHeight`, `ItalicAngle`, `WeightClass`, `WidthClass`, `StemV`, `Flags` 등)
- 가변 글꼴을 정적 마스터로 **인스턴싱**(`Instance`, `InstanceBytes`)하고 글리프 집합으로 **서브셋**(복합 글리프를 닫고 이전→새 ID 맵을 제공하는 `SubsetTrueType`, `seac`를 닫는 `SubsetCFF`)
- 논제로 와인딩 규칙에 따른 4×4 슈퍼샘플링 안티앨리어싱 래스터화

래스터화는 균일한 슈퍼샘플링을 사용하며(네이티브 TrueType/CFF 힌터의 드롭아웃 제어에 따른 델타 힌팅 래스터화가 아님), 아주 작은 크기에서의 마지막 선명함보다 정확성과 이식성을 우선합니다. 이는 HarfBuzz + FreeType이 다루는 경로에서 기능이 완전한 엔진이며, 위의 어떤 항목도 스텁이 아닙니다.

## 테스트 {#testing}

테스트는 100% 커버리지에 도달하기 위해 외부 글꼴에 의존하지 않습니다. 최소한이지만 유효한 TrueType 및 CFF 글꼴(테이블 디렉터리 + `head`/`maxp`/`hhea`/`hmtx`/`cmap`/`loca`/`glyf`, 또는 합성 `MATH` 테이블)을 메모리에서 합성해, 오류 경로를 포함한 모든 파싱 및 래스터화 분기를 결정적으로 실행합니다. 실제 글꼴, 즉 Adobe의 Source Serif 4와 `MATH` 테이블을 위한 STIX Two Math(둘 다 SIL Open Font License이며 `testdata/` 아래에 포함)는 `example_test.go`와 몇 가지 건전성 점검 테스트에서 추가로 처음부터 끝까지 실행되므로, 문서화된 예제가 실제 글꼴 데이터에 대한 스모크 테스트를 겸합니다. CI는 **100.0% 구문 커버리지**, `go vet`, 그리고 `linux/{amd64,arm64,riscv64,loong64,ppc64le,s390x}`, `js/wasm`, `darwin/arm64`, `windows/amd64`에 대한 크로스 컴파일 스모크를 강제합니다.

소스: [github.com/go-opentype/opentype](https://github.com/go-opentype/opentype) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/opentype)
