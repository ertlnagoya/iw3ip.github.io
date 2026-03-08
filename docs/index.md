# IoTxWeb3 Intelligence Platform (IW3IP) ドキュメント

このサイトは、ワークショップ運営と実習手順を分けて管理するためのドキュメントです。

## 想定読者

- コンピュータを学ぶ大学生（学部2-4年）
- 演習を担当する講師・TA

初学者は、まず [学習基礎 / 授業ガイド](foundations/course-guide.md) から読むことを推奨します。

## このサイトだけで学べる範囲

このサイトは、**基礎理解から基本ハンズオンまでをサイト内だけで完結**できるように構成しています。

- 基礎学習:
  - ブロックチェーン
  - Hardhat
  - SSI / DID / VC
- 実践:
  - 最短起動
  - 各 Hands-on サンプル
  - トラブルシュート

外部サイト・論文は、**さらに深く学びたい場合の補助資料**として最後に参照する位置付けです。

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

## 推奨学習順

1. [授業ガイド](foundations/course-guide.md)
2. [学習ロードマップ](foundations/roadmap.md)
3. [ブロックチェーン基礎](foundations/blockchain-basics.md)
4. [Hardhat基礎](foundations/hardhat-basics.md)
5. [SSI/DID/VC基礎](foundations/ssi-did-vc-basics.md)
6. [最短起動](workshop/quickstart.md)
7. [Hands-on](hands-on/index.md)
8. [参考文献](foundations/references.md)

## ローカルでDocsを起動

```bash
pip install mkdocs-material mkdocs-static-i18n
mkdocs serve
```

- URL: `http://127.0.0.1:8000`
