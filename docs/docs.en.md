# Docs (MkDocs)

This website is built with MkDocs [1]. It uses Material for MkDocs [2] as the theme and mkdocs-static-i18n [3] for switching between Japanese and English.

## Who this page is for

- people previewing the site locally for the first time
- people editing and publishing documentation

## Local preview

First, install dependencies.

```bash
python3 -m venv .venv
source .venv/bin/activate
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

- <https://www.ertl.jp/iw3ip.github.io/>

## References

[1] MkDocs, "MkDocs," <https://www.mkdocs.org/>

[2] M. Donath, "Material for MkDocs," <https://squidfunk.github.io/mkdocs-material/>

[3] Ultrabug, "mkdocs-static-i18n," <https://ultrabug.github.io/mkdocs-static-i18n/>
