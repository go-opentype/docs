# go-opentype docs

Source for **<https://go-opentype.github.io/docs/>** — MkDocs Material,
versioned with [mike](https://github.com/jimporter/mike).

Preview locally:

```sh
pip install -r requirements.txt
mkdocs serve
```

Build:

```sh
mkdocs build
```

Deployment runs on push to `main` (`.github/workflows/docs.yml`): it
`mike deploy`s to the `gh-pages` branch, which GitHub Pages serves.

## License

BSD-3-Clause.
