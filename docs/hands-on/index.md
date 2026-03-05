# Hands-on（実習手順）

この章は、参加者がそのまま実行できるレシピ集です。

## 学習目標（大学生向け）

- 実行ログを根拠に、システム挙動を説明できる
- `allowed` / `denied` の差を policy 条件で説明できる
- データフロー上のオンチェーン/オフチェーン境界を説明できる

## 進め方

1. どちらかの入力ソースを選ぶ
   - [HUSKYLENS2サンプル](huskylens2.md)
   - [USBウェブカメラサンプル](webcam.md)
   - [HA x SSI Publisherサンプル](ha-ssi-publisher.md)
2. 選んだサンプルの成功判定を確認する
3. 結果を共有し、必要に応じて次のサンプルへ進む

## 共通の成功判定

- カメラ系（HUSKYLENS2 / Webcam）:
  - `mediator-owner/raw_data/output` にイベントファイルができる
  - フロントに商品が表示され、購入できる
- HA x SSI Publisher系:
  - `/simulate/publish` で `allowed` / `denied` の両ケースを確認できる
  - `/audit/logs` に `allow` / `deny` / `send_error` が記録される
