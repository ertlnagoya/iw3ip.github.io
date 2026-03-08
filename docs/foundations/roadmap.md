# 学習ロードマップ（初級者向け）

このロードマップは、IW3IPを「動かす」だけでなく「説明できる」状態を目指すための優先度付き計画です。

どのページから読めばよいか迷ったときは、この章を基準にすると全体を見失いにくくなります。  
短時間で流れをつかむ場合も、数週間かけて理解を深める場合も、同じ順序で学べるように並べています。

## 優先順位

| 優先度 | テーマ | 目標 | 目安時間 |
|---|---|---|---|
| P0 | サイト内導線の確認 | どの順で読めばよいかを把握する | 30分 |
| P0 | Hands-on再現 | `allow/deny` を再現し監査ログを読める | 2-3時間 |
| P1 | ブロックチェーン基礎 + Hardhat | ローカルチェーンの役割を説明できる | 3-4時間 |
| P1 | SSI/DID/VC基礎 | Consent VC判定の意味を説明できる | 3-4時間 |
| P2 | 発展（イベント共有・PEP） | Phase 2/3の設計意図を説明できる | 2-3時間 |

## 最短学習パス

1. [授業ガイド](course-guide.md)
2. [ブロックチェーン基礎](blockchain-basics.md)
3. [Hardhat基礎](hardhat-basics.md)
4. [SSI/DID/VC基礎](ssi-did-vc-basics.md)
5. [最短起動](../workshop/quickstart.md)
6. [HA x SSI Publisherサンプル](../hands-on/ha-ssi-publisher.md)

この6ページで、基本理解と最小ハンズオンは完結します。

## 3週間プラン（例）

| 週 | 学習内容 | 具体タスク | 成果物 |
|---|---|---|---|
| Week 1 | 実行と観察 | HA x SSIサンプルを実行し、`allowed/denied/send_error` を意図的に発生 | 実行ログ + 気づきメモ |
| Week 2 | 基礎理解 | Blockchain/Hardhat、SSI/DID/VC基礎ページを学習し、用語を整理 | 用語集（自分用） |
| Week 3 | 説明と改善 | なぜこの設計かを説明し、1つ改善PRを作成 | PR 1本 |

## 最小チェックリスト

- [ ] `docker compose up` で環境を起動できる
- [ ] Consent VC登録前後で `denied -> allowed` に変化することを確認した
- [ ] `dataset_id` と `purpose` が判定キーであることを説明できる
- [ ] Hardhat が「開発用ローカルチェーン」であることを説明できる
- [ ] DID/VC が「誰がどの条件でアクセスできるか」を担うことを説明できる

## 次に読むページ

1. [ブロックチェーン基礎](blockchain-basics.md)
2. [Hardhat基礎](hardhat-basics.md)
3. [SSI/DID/VC基礎](ssi-did-vc-basics.md)
4. [参考文献](references.md)

## 外部資料の使い方

- まずは本サイト内の説明を優先してください。
- 外部資料は、用語の原典確認、詳細仕様の確認、追加調査のために使います。
