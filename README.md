# go-opentype docs

Source for **<https://go-opentype.github.io/docs/>** — Hugo with the
[tannevaled/hextra](https://github.com/tannevaled/hextra) fork (`themes/hextra`,
a submodule pinned at a release tag). It was MkDocs Material with
[mike](https://github.com/jimporter/mike) until the migration to Hugo; every
page keeps its URL.

## Languages

English is the default and sits at the root of each version. Simplified Chinese
(`/zh-cn/`), Korean (`/ko/`) and Japanese (`/ja/`) are under their code, with
the same file names under `content/<lang>/`; a translated heading keeps the
English one's explicit anchor. The CJK pages rely on
`markup.goldmark.extensions.cjk` (`eastAsianLineBreaks`, `css3draft`, not
`simple`, which glues Korean words together) and on the system CJK font stacks
in `assets/css/custom.css`. A change to an English page is a change to its three
translations too.

Preview locally:

```sh
git submodule update --init --recursive   # themes/hextra
hugo server                               # http://localhost:1313
```

Build:

```sh
hugo --gc --minify --panicOnWarning
```

Deployment runs on push to `main` (`.github/workflows/docs.yml`): it builds
the version named by `params.docs.version` in `hugo.yaml`, checks the links, and
publishes it to the `gh-pages` branch, which GitHub Pages serves.

## License

BSD-3-Clause.
