# Hands-on（実習手順）

この章は、参加者がそのまま実行できるレシピ集です。

## 学習目標（大学生向け）

- 実行ログを根拠に、システム挙動を説明できる
- `allowed` / `denied` の差を policy 条件で説明できる
- データフロー上のオンチェーン/オフチェーン境界を説明できる

## この章の位置付け

この章は、基礎章を読んだあとに「実際に動かして確認する」ための章です。  
基本的な実行と確認はこのサイト内の説明だけで完結し、外部資料は必須ではありません。

各ページには、対応する `問題用プログラム`、`解答用プログラム`、`演習説明` へのリンクも追加しています。  
ワークショップでは、まず問題用を配布し、詰まった場合に解答用や演習説明へ進む使い方を想定しています。

## 進め方

1. どちらかの入力ソースを選ぶ
   - [HUSKYLENS2サンプル](huskylens2.md)
   - [USBウェブカメラサンプル](webcam.md)
   - [USBウェブカメライベント共有サンプル（Phase 2）](webcam-event-sharing.md)
   - [HA x SSI Publisherサンプル](ha-ssi-publisher.md)
   - [環境・防災イベント共有サンプル（Phase 2）](environment-disaster.md)
   - [地域安全アシスタントサンプル（Phase 3）](regional-safety-assistant.md)
   - [LLM Plannerハンズオン](llm-planner.md)
   - [LLM Planner置き換え仕様](llm-planner-spec.md)
2. 選んだサンプルの成功判定を確認する
3. 結果を共有し、必要に応じて次のサンプルへ進む

## 共通の成功判定

- カメラ系（HUSKYLENS2 / Webcam）:
  - `mediator-owner/raw_data/output` にイベントファイルができる
  - フロントに商品が表示され、購入できる
- カメラ系 Phase 2:
  - `home/event/possible_littering` をイベント共有として再現できる
  - `community_cleaning` と `advertising` で `allowed` / `denied` を確認できる
- HA x SSI Publisher系:
  - `/simulate/publish` で `allowed` / `denied` の両ケースを確認できる
  - `/audit/logs` に `allow` / `deny` / `send_error` が記録される
- 非カメラ Phase 2 系:
  - `home/event/flood_risk_high` をイベント共有として再現できる
  - `disaster_response` と `advertising` で `allowed` / `denied` を確認できる
- Phase 3 系:
  - 自然言語要求から `plan` を生成できる
  - 条件成立時に `light_on` と `send_notification` が `executed` になる
  - `/assistant/executions` で履歴を確認できる
- Phase 3 発展仕様:
  - `stub` provider で LLM planner を再現できる
  - OpenAI 互換 API 向けの環境変数を説明できる
  - rule-based planner を LLM planner に差し替える設計方針を説明できる
  - 構造化出力、許可イベント、許可アクションの制約を説明できる
