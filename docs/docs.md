# Docs (MkDocs)

このWebサイトは MkDocs で構築されています。

## このページの対象

- 初めてサイトをローカルで表示したい人
- ドキュメントを編集して反映したい人

利用している主なツール:

- MkDocs: <https://www.mkdocs.org/>
- Material for MkDocs: <https://squidfunk.github.io/mkdocs-material/>
- mkdocs-static-i18n: <https://ultrabug.github.io/mkdocs-static-i18n/>

## ローカル確認

まず依存を入れます。

```bash
pip install -r docs/requirements.txt
mkdocs serve
```

表示URL:

- <http://127.0.0.1:8000>

## ビルド

```bash
mkdocs build --strict
```

`--strict` は、リンクや設定の問題を早めに見つけるために有効です。

## 公開

- GitHub Pages
- Workflow: `.github/workflows/docs-pages.yml`

公開先:

- <https://ertlnagoya.github.io/iw3ip.github.io/>
