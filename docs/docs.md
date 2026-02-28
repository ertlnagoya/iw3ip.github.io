# Docs (MkDocs)

このWebサイトは MkDocs で構築されています。

## ローカル確認

```bash
pip install -r docs/requirements.txt
mkdocs serve
```

## ビルド

```bash
mkdocs build --strict
```

## 公開

- GitHub Pages
- Workflow: `.github/workflows/docs-pages.yml`
