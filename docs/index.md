# IoTxWeb3 Intelligence Platform (IW3IP) ドキュメント

このサイトは、IW3IP の全体像、実装例、ハンズオン手順を段階的にたどれるように構成したドキュメントです。

## 主な利用場面

- コンピュータや情報システムを学ぶ学部学生
- 演習を担当する講師・TA

[学習基礎 / 授業ガイド](foundations/course-guide.md) から読むと、全体の見取り図をつかみやすくなります。

## このサイトでたどれる内容

このサイトでは、基礎理解から基本ハンズオンまでを、順にたどりながら一通り確認できるようにしています。

- 基礎学習:
  - ブロックチェーン
  - Hardhat
  - SSI / DID / VC
- 実践:
  - 最短起動
  - 各 Hands-on サンプル
  - トラブルシュート

外部サイトや論文は、標準仕様や研究背景をさらに追いたいときに参照する資料として整理しています。

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

## まず 1 つ動かしたい場合

Phase 3 の最短デモとして、`assistant-demo` を使うと 1 コマンドで次をまとめて起動できます。

- `assistant-demo`
- `llm-mock`
- `assistant-ui`

対応ページ:

- [LLM Plannerハンズオン](hands-on/llm-planner.md)

起動コマンド:

```bash
docker compose -f infra/docker-compose.yml --profile assistant-demo up --build -d
```

開く URL:

- `http://localhost:4173`

## 推奨学習順

1. [授業ガイド](foundations/course-guide.md)
2. [学習ロードマップ](foundations/roadmap.md)
3. [ブロックチェーン基礎](foundations/blockchain-basics.md)
4. [Hardhat基礎](foundations/hardhat-basics.md)
5. [SSI/DID/VC基礎](foundations/ssi-did-vc-basics.md)
6. [最短起動](workshop/quickstart.md)
7. [Hands-on](hands-on/index.md)
8. [参考文献](foundations/references.md)

## 研究室に興味のある方へ

研究室でこの分野に取り組みたい方は、次の所属先を希望してください。

- 学部生:
  - 情報学部コンピュータ科学科 情報システム系
- 大学院生:
  - 情報学研究科 情報システム学専攻
  - 高田・松原研究室

<span style="color: #c62828; font-weight: 700;">研究生は基本的には受け入れていません。</span>

博士号の取得を目指したい方は、博士後期課程への入学を相談してください。

大学院入試の詳細は、次を参照してください。

- 日本語: <https://www.i.nagoya-u.ac.jp/gs/entranceexamination/>
- 英語: <https://www.i.nagoya-u.ac.jp/en/gs/entranceexamination/>

## ローカルでDocsを起動

```bash
pip install mkdocs-material mkdocs-static-i18n
mkdocs serve
```

- URL: `http://127.0.0.1:8000`
