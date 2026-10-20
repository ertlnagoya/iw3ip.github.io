# Docs (MkDocs)

このWebサイトは MkDocs [1] で構築されています。テーマには Material for MkDocs [2]、日本語と英語の切り替えには mkdocs-static-i18n [3] を使っています。

## このページの対象

- 初めてサイトをローカルで表示したい人
- ドキュメントを編集して反映したい人

## ローカル確認

依存パッケージをインストールし、ローカルサーバを起動します。

```bash
python3 -m venv .venv
source .venv/bin/activate
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

- <https://www.ertl.jp/iw3ip.github.io/>

## 参考文献

[1] MkDocs, "MkDocs," <https://www.mkdocs.org/>

[2] M. Donath, "Material for MkDocs," <https://squidfunk.github.io/mkdocs-material/>

[3] Ultrabug, "mkdocs-static-i18n," <https://ultrabug.github.io/mkdocs-static-i18n/>
