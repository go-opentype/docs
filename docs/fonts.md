# fonts — bundled legible families

`github.com/go-opentype/fonts` provides 44 legible, permissively-licensed
TrueType fonts, one Go subpackage per family, each with its own
`//go:embed` — no downloading, no sourcing a `.ttf` yourself, and **your
binary links only the families you import**. Coverage spans Latin,
Cyrillic and Greek plus seven non-Latin scripts — Arabic and Hebrew (RTL),
Devanagari, Thai, Georgian, Armenian and Egyptian Hieroglyphs — plus Noto
Sans SC for CJK (Han ideographs, kana and common CJK punctuation). Built
for [opentype](opentype.md), the pure-Go, stdlib-only TrueType engine, but
the raw bytes work with any parser that accepts a `.ttf`.

```go
import (
	"github.com/go-opentype/fonts"
	"github.com/go-opentype/fonts/inter"
)

f, err := fonts.Parse(inter.TTF) // *opentype.Font
face := f.NewFace(16)            // 16px face
```

Pure Go, no cgo, no external assets to fetch at build or run time — every
family is `//go:embed`ded into its own subpackage.

## The lazy-at-compile-time import model

`//go:embed` is eager *per package*: any package that embeds a font links
that font's bytes into every binary that imports it, whether or not the
binary ever uses it. A `fonts` package that bulk-embedded all 44 families
would put all 44 into your binary the moment you imported it for anything
at all.

So the root `fonts` package doesn't do that. Each family lives in its own
subpackage — `fonts/inter`, `fonts/roboto`, `fonts/jetbrainsmono`, and so
on — with its own `//go:embed`. Importing `fonts/inter` links only Inter.
Importing ten subpackages links only those ten.

The root `fonts` package holds two things only:

1. **[`Family`](#api) metadata** — name, [`Kind`](#api), license, and import
   path for every bundled font, returned by `All` and `ByName`. No
   `[]byte` field: enumerating families never links any of them in.
2. **[`MostLegible`](#api)** — the one family embedded directly in the root
   package, so there's a sensible zero-extra-imports default. Every other
   family requires importing its own subpackage.

## Bundled fonts

Six families are hand-curated (present since v0.1.0); the other thirty-eight
were ingested from [google/fonts](https://github.com/google/fonts). Seven
non-Latin families (Arabic, Hebrew, Devanagari, Thai, Georgian, Armenian,
Egyptian Hieroglyphs) and Noto Sans SC (the bundled CJK family) round out
script coverage beyond Latin/Cyrillic/Greek.

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
| … 30 more Latin/Cyrillic/Greek families (Roboto, Open Sans, Montserrat, Fira Sans/Code, IBM Plex, …) | Sans/Serif/Mono | OFL-1.1 | see [pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts) for the full list |

Full license texts are bundled verbatim under `licenses/` — one file per
family (SIL Open Font License 1.1 for every `OFL-1.1` row, plus
`GoFonts-LICENSE.txt` — BSD-3-Clause, Bigelow & Holmes — shared by Go and
Go Mono).

!!! success "The default: Atkinson Hyperlegible"
    Designed by the Braille Institute specifically to maximize character
    distinction for readers with low vision, benefiting every reader in the
    process. [`MostLegible()`](#api) returns it, and it's the only family
    embedded directly in the root package.

!!! info "Bundled instances are static; the engine itself supports variable fonts"
    **Inter**, **Lora**, **JetBrains Mono**, and most of the
    `google/fonts`-ingested set are variable fonts upstream; each bundled
    `.ttf` is pinned at that family's default master (static instance) to
    keep the bundle simple and small. That's a packaging choice, not an
    engine limitation — [opentype](opentype.md#variable-fonts) fully
    supports OpenType Variations via `(*Face).SetVariation`. If you need a
    specific axis coordinate (e.g. Inter at weight 600), source that
    family's variable `.ttf` yourself and parse it with `opentype.Parse`.

## API

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

Every other family's bytes live in its own subpackage as `var TTF []byte`,
e.g. `inter.TTF`, `roboto.TTF`, `jetbrainsmono.TTF` — see the table above
and [pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts) for the
full list of import paths.

## License

The Go source code in the repository (everything outside the bundled
`.ttf` files and `licenses/`) is BSD-3-Clause. The bundled font files keep
their own upstream licenses; see the table above and `licenses/` for the
full texts, copyright notices, and (for the OFL fonts) their Reserved Font
Names.

Source: [github.com/go-opentype/fonts](https://github.com/go-opentype/fonts) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts)
