# go-opentype docs

Source for **<https://go-opentype.github.io/docs/>** — Hugo with the
[tannevaled/hextra](https://github.com/tannevaled/hextra) fork (`themes/hextra`,
a submodule pinned at a release tag). It was MkDocs Material with
[mike](https://github.com/jimporter/mike) until the migration to Hugo; every
page keeps its URL.

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
