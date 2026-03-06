# Docs (MkDocs)

This website is built with MkDocs.

## Who this page is for

- people previewing the site locally for the first time
- people editing and publishing documentation

Main tools used here:

- MkDocs: <https://www.mkdocs.org/>
- Material for MkDocs: <https://squidfunk.github.io/mkdocs-material/>
- mkdocs-static-i18n: <https://ultrabug.github.io/mkdocs-static-i18n/>

## Local preview

First, install dependencies.

```bash
pip install -r docs/requirements.txt
mkdocs serve
```

Open:

- <http://127.0.0.1:8000>

## Build

```bash
mkdocs build --strict
```

`--strict` is recommended so that link/configuration issues are detected early.

## Publish

- GitHub Pages
- Workflow: `.github/workflows/docs-pages.yml`

Published URL:

- <https://ertlnagoya.github.io/iw3ip.github.io/>
