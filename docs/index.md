# IoTxWeb3 Intelligence Platform (IW3IP) ドキュメント

このサイトは、ワークショップ運営と実習手順を分けて管理するためのドキュメントです。

## 想定読者

- コンピュータを学ぶ大学生（学部2-4年）
- 演習を担当する講師・TA

初学者は、まず [学習基礎 / 授業ガイド](foundations/course-guide.md) から読むことを推奨します。

## ドキュメントの考え方

- **Workshop**: 講師・TA向けの進行設計（目的、時間配分、運営）
- **Hands-on**: 受講者向けの実作業手順（コマンド、期待結果、確認ポイント）

つまり、**Workshop の一部として Hands-on を実施する**構成です。

## 全体フロー

![全体イメージ](assets/raspberryPi.jpg)

## はじめに読むページ

1. [Workshop / 事前準備](workshop/prerequisites.md)
2. [Learning Foundations / 学習ロードマップ](foundations/roadmap.md)
3. [Workshop / 最短起動](workshop/quickstart.md)
4. [Hands-on](hands-on/index.md)
5. [Operations / トラブルシュート](operations/troubleshooting.md)

## ローカルでDocsを起動

```bash
pip install mkdocs-material mkdocs-static-i18n
mkdocs serve
```

- URL: `http://127.0.0.1:8000`
