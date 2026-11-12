# 学習ロードマップ（初級者向け）

IW3IP を「動かせる」だけでなく「説明できる」状態を目指すための学習計画です。

どのページから読むか迷った場合は、この順序に従ってください。短時間で進める場合も数週間かける場合も、進める順序は同じです。サイト内のほかのページに書いた読む順は、このページの短縮版です。

## 優先順位

上の行から順に進めます。順番は、下の最短学習パスと同じです。

| 優先度 | テーマ | 目標 | 目安時間 |
|---|---|---|---|
| 必須 | サイト内導線の確認 | どの順で読めばよいかを把握する | 約 30 分 |
| 必須 | ブロックチェーン基礎 + Hardhat | ローカルチェーンの役割を説明できる | 約 3〜4 時間 |
| 必須 | SSI/DID/VC基礎 | Consent VC判定の意味を説明できる | 約 3〜4 時間 |
| 必須 | Hands-on再現 (事前準備 → 1.1 → 最短起動 → 1.2 → 1.3) | `allow/deny` を再現し監査ログを読める。出品から購入までを通せる | 約 2〜3 時間 |
| 発展 | 発展（イベント共有・PEP（Policy Enforcement Point）） | Phase 2/3の設計意図を説明できる | 約 2〜3 時間 |

## 最短学習パス

1. [ブロックチェーン基礎](blockchain-basics.md)
2. [Hardhat基礎（このサイトで使うブロックチェーン）](hardhat-basics.md)
3. [SSI / DID / VC 基礎](ssi-did-vc-basics.md)
4. [事前準備 (ツールのインストール)](../setup/prerequisites.md)
5. [実機なしで一連を動かす (HA Demo Simulator)](../hands-on/ha-demo-simulator.md)（実機なしで一通り動かす）
6. [最短起動 (スタックを動かす)](../setup/quickstart.md)
7. Part 1 の 1.2 から 1 つ: [AI カメラで人物を検知する (HUSKYLENS2)](../hands-on/huskylens2.md)、[USB カメラでポイ捨てを検知する (OpenCV)](../hands-on/webcam.md)、[Home Assistant のデータを取り込む (HA SSI Publisher)](../hands-on/ha-ssi-publisher.md)
8. Part 1 の 1.3 で出品と購入を行う ([ハンズオン](../hands-on/index.md#13-マーケットプレイスで売買-v1-経路) の 1.3)
9. Part 1 の 1.4 (任意)

7 で [Home Assistant のデータを取り込む (HA SSI Publisher)](../hands-on/ha-ssi-publisher.md) を選ぶ場合は、6 の最短起動を 7 のあと (8 の前) に行います。このページの publisher は、最短起動の IPFS と同じポート 8080 を使うためです。

ここまでで、基本理解と最小ハンズオンは完結します。この先は、[ハンズオン](../hands-on/index.md) の Part 2、Part 3 に必要な分だけ進みます。

## 3週間プラン（例）

| 週 | 学習内容 | 具体タスク | 成果物 |
|---|---|---|---|
| Week 1 | 基礎理解と最初の実行 | Blockchain/Hardhat、SSI/DID/VC基礎ページを学習し、用語を整理する。[事前準備 (ツールのインストール)](../setup/prerequisites.md) を済ませ、[実機なしで一連を動かす (HA Demo Simulator)](../hands-on/ha-demo-simulator.md) を実行する | 用語集（自分用） |
| Week 2 | 実行と観察 | [最短起動 (スタックを動かす)](../setup/quickstart.md)、Part 1 の 1.2 から 1 つ、1.3 の出品と購入を行う。1.2 で [Home Assistant のデータを取り込む (HA SSI Publisher)](../hands-on/ha-ssi-publisher.md) を選んだ場合は、最短起動の前に行い、`allowed/denied/send_error` を意図的に発生させる | 実行ログ + 気づきメモ |
| Week 3 | 説明と改善 | なぜこの設計かを説明し、1つ改善PRを作成 | PR 1本 |

## 最小チェックリスト

- [ ] `docker compose up` で環境を起動できる
- [ ] 同意 (consent) の JSON レコードを `/consents` に登録する前後で、`denied -> allowed` に変化することを確認した
- [ ] `dataset_id` と `purpose` が判定キーであることを説明できる
- [ ] Hardhat が「開発用ローカルチェーン」であることを説明できる
- [ ] DID/VC が「誰がどの条件でアクセスできるか」を担うことを説明できる

## 次に読むページ

1. 最短学習パスの最初のページ: [ブロックチェーン基礎](blockchain-basics.md)
2. 標準仕様や研究の背景を調べたいときは [参考文献・発展学習資料](references.md)
3. ハンズオンのあとで仕組みを詳しく知りたいときは [設計](../design/index.md)

## 外部資料の使い方

- まずサイト内の説明を一通り読む
- 外部資料は用語の原典確認・詳細仕様・追加調査に使う
