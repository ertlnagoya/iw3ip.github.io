# Docs Site (MkDocs, ja/en)

このディレクトリは、`IoTxWeb3 Intelligence Platform (IW3IP)` の
公開サイトコンテンツを管理します。

## 多言語構成

- 既定: 日本語（`*.md`）
- 英語: `*.en.md`
- 方式: `mkdocs-static-i18n`（suffix構成）

## ローカル起動

```bash
pip install -r docs/requirements.txt
mkdocs serve
```

- URL: `http://127.0.0.1:8000`

## ビルド

```bash
mkdocs build --strict
```

## 公開

- GitHub Pages via `.github/workflows/docs-pages.yml`
