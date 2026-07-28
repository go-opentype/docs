# Guide: choose an accessible font

[`fonts`](../fonts.md) bundles 44 families so a Go program can ship legible
text without sourcing or downloading a `.ttf`. This guide is about picking
the right one — not just the default.

## Just want something legible? Use the default

```go
f, err := fonts.Parse(fonts.MostLegible())
```

`MostLegible()` returns **Atkinson Hyperlegible**, designed by the Braille
Institute of America specifically to maximise character distinction for
readers with low vision — the letterforms that are easiest to confuse in
most sans-serifs (`I`/`l`/`1`, `O`/`0`) are deliberately differentiated.
That benefits every reader, not only low-vision ones, which is why it's the
package default rather than an opt-in.

## Picking by role

`fonts.All()` returns every bundled `Family`, each tagged with a `Kind`
(`KindSans`, `KindSerif`, `KindMono`) so you can filter for a role rather
than hardcode a name:

```go
for _, fam := range fonts.All() {
    if fam.Kind == fonts.KindMono {
        // e.g. Go Mono or JetBrains Mono — a code block face
    }
}
```

Or look one up by name to discover its import path (`Family` carries no
`[]byte` — enumerating or searching families never links their bytes in;
you still import the subpackage to reach the bytes themselves):

```go
fam, ok := fonts.ByName("Lora") // case-insensitive
if ok {
    fmt.Println(fam.ImportPath) // github.com/go-opentype/fonts/lora
}
// import "github.com/go-opentype/fonts/lora", then:
f, err := fonts.Parse(lora.TTF)
```

## What's bundled, and why you'd pick it

| Family | Kind | Best for |
| --- | --- | --- |
| **Atkinson Hyperlegible** | Sans | Default choice; body text where legibility matters most |
| Inter | Sans | UI chrome, dense interfaces, a more neutral geometric sans |
| Go | Sans | The Go project's own sans — a natural pick for Go-tooling UIs |
| Lora | Serif | Longer-form reading, editorial contexts |
| Go Mono | Mono | Code, matching the Go sans above |
| JetBrains Mono | Mono | Code, wider ligature-friendly monospace |

## The variable-font caveat

Inter, Lora, and JetBrains Mono ship as **variable fonts** upstream (a
single file spanning a weight/width axis range). `go-opentype/fonts` pins
each to its **default static instance** — the `.ttf` bundled here is one
fixed weight, not the full variable range — to keep the bundle simple and
small, not because the engine can't handle axes:
[`opentype` fully supports OpenType Variations](../opentype.md#variable-fonts),
including `SetVariation` to instance any axis coordinate. If your design
needs a specific weight along an axis (e.g. Inter at 600 rather than its
default 400), source that family's variable `.ttf` yourself — from
[google/fonts](https://github.com/google/fonts), for instance — and drive it
with `opentype.Parse` + `(*Face).SetVariation(map[string]float64{"wght": 600})`.

## License obligations

The Go source in `fonts` is BSD-3-Clause, but the **font files themselves**
keep their upstream licenses — OFL-1.1 for Atkinson Hyperlegible, Inter,
Lora, and JetBrains Mono; BSD-3-Clause for Go/Go Mono. If you redistribute a
binary that embeds these fonts, carry the corresponding license text from
`licenses/` along with it; see [fonts](../fonts.md#bundled-fonts) for the
full table and copyright lines.
