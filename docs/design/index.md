# 設計

「設計」の章には、IW3IP の各機能を**どのような仕組みで作り、なぜその方式を選んだか**を定める設計仕様をまとめています。[ハンズオン](../hands-on/index.md) が「手順どおりに動かして確かめる」章であるのに対し、設計は「仕組みと判断の理由を説明する」章です。手順やコマンドは書かず、VC とトークンの種類、API、データの流れ、検討した選択肢、未解決の課題を扱います。

ハンズオンを終えたあとに「なぜこう動くのか」を知りたいとき、機能を拡張ないしは実装するとき、研究として発展させるときに読みます。ハンズオンを進めるだけなら、読まなくても構いません。

## サイトの各章の違い

| 章 | 目的 | 書いてあること | 主な読者 | 読むとき |
|---|---|---|---|---|
| [ハンズオン](../hands-on/index.md) | IW3IP を動かして理解する | コマンド、期待する出力、確認ポイント | 受講者 | 演習の時間に、順番に |
| 設計 (この章) | 仕組みと設計判断を説明する | VC・トークン・API・データの流れ、選んだ方式と理由、未解決の課題 | 機能を拡張・実装する人、発展学習をする受講者、講師 | ハンズオンのあと、ないしは機能を変えるとき |
| [困ったとき](../operations/index.md) | 演習中に詰まったときに解決する | 症状ごとの確認と対処、よくある質問 | 受講者・TA | 手順どおりに動かないとき |
| [授業ガイド](../foundations/course-guide.md) | 授業を組み立てて進める | コマ数別の構成、評価方法、進行の時間配分 | 講師・TA | 授業の準備をするとき |

## 設計仕様の一覧

| 設計仕様 | 扱う範囲 | 対応するハンズオン |
|---|---|---|
| [VC アーキテクチャ全体像 (Phase 2 / Stage 0〜7, T, A, P)](vc-architecture-overview.md) | 6 種類の VC と 4 種類のトークン、どの API をどの VC で認可するか、Stage の一覧 | Part 2 全体 |
| [Marketplace VC Bridge — v1 / v2 設計仕様](marketplace-vc-bridge-spec.md) | マーケットプレイスと SSI ウォレットの接続 (v1 / v2) | [購入後に証明書をウォレットへ渡す (Bridge / Stage 5)](../hands-on/marketplace-vc-bridge.md)、[ServiceVC で書いたデータを PurchaseViewerVC で読み出す (Stage 6)](../hands-on/marketplace-vc-end-to-end.md) |
| [SellerVC — マーケット出品ガバナンス VC (Stage 7 設計仕様)](seller-vc-spec.md) | 出品者の身元の確認 | [出品者の身元を裏付ける (Seller VC / Stage 7)](../hands-on/marketplace-seller-vc.md) |
| [段階アクセスの設計仕様](../hands-on/data-user-vc-tiered-spec.md) | 受信者の信頼度に応じた出し分け (DataUserVC、VLM による派生データ) | [信頼度に応じて見せる中身を変える (DataUserVC / Stage T)](../hands-on/data-user-vc-tiered.md) から始まる Stage T の 3 ページ |
| [C2PA 来歴パイプライン — カメラ映像とセンサーデータの来歴証明 (Stage P 設計仕様)](c2pa-provenance-spec.md) | 画像の来歴の記録と署名、publisher での検証、チェーンへの記録、VLM の要約の来歴 (P0〜P3 を実装済み) | [画像の来歴を証明する (C2PA / Stage P)](../hands-on/c2pa-provenance.md)、[来歴を publisher とチェーンにつなぐ (C2PA / Stage P の P1・P2)](../hands-on/c2pa-provenance-publisher.md)、[VLM の要約から元の画像をたどる (C2PA / Stage P の P3)](../hands-on/c2pa-provenance-vlm.md) |
| [LLM Planner 置き換え仕様](../hands-on/llm-planner-spec.md) | 地域安全アシスタントの planner を LLM に置き換えるときの設計の境界 | [AI に手順を考えさせる (LLM Planner)](../hands-on/llm-planner.md) |

## 設計仕様の読み方

- 多くの仕様は実装の前に書いた設計です。実装と違う箇所には、その旨の注記を付けています (「→ 実装では」の行、「当初の設計では／現在の実装では」の書き分け、「現在の実装」の囲みなど)。実際の動作は、教材リポジトリのソースコードとハンズオンの手順を正とします。
- 本文中の Stage は、Phase 2 のハンズオンの段階番号です (一覧は [VC アーキテクチャ全体像 (Phase 2 / Stage 0〜7, T, A, P)](vc-architecture-overview.md) の §5)。Part、Phase、Stage の関係は [ハンズオン](../hands-on/index.md#part-phase-stage) で説明しています。
