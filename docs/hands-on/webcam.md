# USBウェブカメラサンプル（ポイ捨て検知）

## 目的

HUSKYLENS2がなくても、USBウェブカメラだけで
`person_detected` / `possible_littering` イベントを生成します。

## 公式リンク

- OpenCV: <https://opencv.org/>
- USBカメラ一般説明（参考）: <https://en.wikipedia.org/wiki/Webcam>

## 前提

- USBウェブカメラ、または mock 実行環境がある
- `webcam-bridge` が利用できる
- `mediator-owner` が起動済みで、`raw_data/output` を監視している
- 実機モードでは、PCからカメラデバイスが認識されている

## 演習用プログラム

- [問題用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/webcam_littering_mock/problem_program.py)
- [解答用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/webcam_littering_mock/answer_program.py)
- [演習説明](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/webcam_littering_mock/README.md)

この演習では、USBウェブカメラの mock 出力として `possible_littering` イベントファイルを生成します。  
問題用プログラムでは、イベントの最小構造と、`camera_id`・`confidence`・`event_type` の意味を確認できます。

## 1. mockで最小確認

```bash
cd webcam-bridge
python3 webcam_litter_bridge.py \
  --mode mock \
  --camera-id 401 \
  --output-dir ../mediator-owner/raw_data/output \
  --flush-seconds 5
```

## 2. 実機（USB webcam）

```bash
python3 webcam_litter_bridge.py \
  --mode webcam \
  --camera-index 0 \
  --camera-id 401 \
  --output-dir ../mediator-owner/raw_data/output \
  --litter-classes bottle,cup \
  --linger-seconds 8 \
  --person-away-seconds 5
```

## 3. 判定ロジック（簡易）

- 人を検出 → `person_detected`
- `bottle/cup` が一定時間残留し、人が近くにいない → `possible_littering`

## 4. 確認ポイント

- `*_webcam_event_*.txt` が生成される
- 商品化されて購入可能になる

## 成功例

- mock モードでイベントファイル生成まで確認できる
- 実機モードで `person_detected` または `possible_littering` が出力される
- 商品一覧に反映され、購入できる

## 5. 注意

この検知は学習用のヒューリスティックであり、厳密な判定ではありません。

## トラブル時

- 症状: カメラが開けない
  - 確認: 別アプリがカメラを占有していないか
  - 確認: `--camera-index 0` を `1` などに変えて試したか
- 症状: 期待イベントが出ない
  - 確認: 照明条件や画角が適切か
  - 確認: mock モードでパイプライン自体が動くか
- 症状: 商品化されない
  - 確認: `mediator-owner` が起動しているか
  - 確認: 出力先パスが正しいか
