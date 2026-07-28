# fonts — bundled legible families

`github.com/go-opentype/fonts` provides six legible, permissively-licensed
TrueType fonts, embedded as `[]byte` payloads with a two-line API — no
downloading, no sourcing a `.ttf` yourself. Built for
[opentype](opentype.md), the pure-Go, stdlib-only TrueType engine, but the
raw bytes work with any parser that accepts a `.ttf`.

```go
import "github.com/go-opentype/fonts"

f, err := fonts.Parse(fonts.MostLegible()) // *opentype.Font
face := f.NewFace(16)                      // 16px face
```

Pure Go, no cgo, no external assets to fetch at build or run time — the
fonts are `//go:embed`ded into the module.

## Bundled fonts

| Name | Kind | License | Copyright | Upstream |
| --- | --- | --- | --- | --- |
| Atkinson Hyperlegible | Sans | OFL-1.1 | Copyright 2020 Braille Institute of America, Inc. | <https://brailleinstitute.org/freefont> |
| Inter | Sans | OFL-1.1 | Copyright 2020 The Inter Project Authors | <https://github.com/rsms/inter> |
| Go | Sans | BSD-3-Clause | Copyright (c) 2016 Bigelow & Holmes Inc. All rights reserved. | <https://go.dev/blog/go-fonts> |
| Lora | Serif | OFL-1.1 | Copyright 2011 The Lora Project Authors, with Reserved Font Name "Lora" | <https://github.com/cyrealtype/Lora-Cyrillic> |
| Go Mono | Mono | BSD-3-Clause | Copyright (c) 2016 Bigelow & Holmes Inc. All rights reserved. | <https://go.dev/blog/go-fonts> |
| JetBrains Mono | Mono | OFL-1.1 | Copyright 2020 The JetBrains Mono Project Authors | <https://github.com/JetBrains/JetBrainsMono> |

Full license texts are bundled verbatim under `licenses/`:
`AtkinsonHyperlegible-OFL.txt`, `Inter-OFL.txt`, `Lora-OFL.txt`,
`JetBrainsMono-OFL.txt` (SIL Open Font License 1.1) and
`GoFonts-LICENSE.txt` (BSD-3-Clause, Bigelow & Holmes).

!!! success "The default: Atkinson Hyperlegible"
    Designed by the Braille Institute specifically to maximize character
    distinction for readers with low vision, benefiting every reader in the
    process. [`MostLegible()`](#api) returns it.

!!! warning "No variable-font support"
    **Inter**, **Lora**, and **JetBrains Mono** are variable fonts upstream;
    the bundled `.ttf` files are pinned at each family's default master
    (static instance). `go-opentype` has no variable-font support (see
    [opentype](opentype.md#support-matrix)), so it always renders that
    default instance — OpenType Variations axes are not applied.

## API

```go
type Kind int

const (
	KindSans Kind = iota
	KindSerif
	KindMono
)

type Family struct {
	Name    string // e.g. "Atkinson Hyperlegible"
	Kind    Kind
	License string // SPDX id: "OFL-1.1" or "BSD-3-Clause"
	TTF     []byte // raw .ttf bytes
}

func AtkinsonHyperlegible() []byte
func Inter() []byte
func GoRegular() []byte
func Lora() []byte
func GoMono() []byte
func JetBrainsMono() []byte

func MostLegible() []byte                    // == AtkinsonHyperlegible()
func All() []Family                          // every bundled family, stable order
func ByName(name string) ([]byte, bool)      // case-insensitive lookup by Family.Name
func Parse(ttf []byte) (*opentype.Font, error) // convenience wrapper over opentype.Parse
```

## License

The Go source code in the repository (everything outside `ttf/` and
`licenses/`) is BSD-3-Clause. The bundled font files keep their own
upstream licenses; see the table above and `licenses/` for the full texts,
copyright notices, and (for the OFL fonts) their Reserved Font Names.

Source: [github.com/go-opentype/fonts](https://github.com/go-opentype/fonts) ·
[pkg.go.dev](https://pkg.go.dev/github.com/go-opentype/fonts)
