# USBウェブカメラサンプル（ポイ捨て検知）

## 目的

HUSKYLENS2がなくても、USBウェブカメラだけで
`person_detected` / `possible_littering` イベントを生成します。

## このページで分かること

- USBウェブカメラだけで Phase 1 のイベント生成を試す方法
- `person_detected` と `possible_littering` をどのように使い分けるか
- 商品化につながる最小の確認ポイント

## つまずきやすい点

- カメラのインデックスや占有状況で最初に失敗しやすい
- 照明や画角でヒューリスティックの出力が大きく変わる
- 検知が動いていても `mediator-owner` 側まで届いていないことがある

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

## 最短ルート

最初は次の 4 手順で十分です。

1. `mock` モードで `webcam_litter_bridge.py` を起動する
2. `*_webcam_event_*.txt` が生成されることを確認する
3. `mediator-owner` が商品化することを確認する
4. フロントで商品一覧に反映されることを確認する

その後の分岐:

- まずパイプラインだけ確認したい場合: `mock` モードで十分です
- 実機のカメラ検知まで見たい場合: `webcam` モードへ進みます
- カメラが開けない場合: `camera-index` と占有状況の確認を先に行います

## 工程別の目次

<details class="iw3ip-toc-details" open>
  <summary>確認 1: mock モードでイベント生成を確認する</summary>
  <p>最初に mock モードでイベントファイル生成を確認し、カメラ実機がなくても後段が動くことを確認します。</p>
  <ol>
    <li><a href="#1-mockで最小確認">mockで最小確認</a></li>
    <li><a href="#4-確認ポイント">確認ポイント</a></li>
    <li><a href="#成功例">成功例</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 2: 実機 webcam モードで検知を確認する</summary>
  <p>次に USB ウェブカメラ実機を使い、`person_detected` と `possible_littering` がどう出力されるかを確認します。</p>
  <ol>
    <li><a href="#2-実機usb-webcam">実機（USB webcam）</a></li>
    <li><a href="#3-判定ロジック簡易">判定ロジック（簡易）</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 3: 注意点とトラブルを整理する</summary>
  <p>最後に、ヒューリスティックの限界と、カメラデバイスや出力先パスで発生しやすいトラブルを整理します。</p>
  <ol>
    <li><a href="#5-注意">注意</a></li>
    <li><a href="#トラブル時">トラブル時</a></li>
  </ol>
</details>

## 読み進め方

Phase 1 のカメラ入力ページです。HUSKYLENS2 と同様、まず `mock` でパイプラインを確認してから実機カメラに進む方が切り分けが楽です。

## Phase 1: mock モードでパイプラインを確認する

## 1. mockで最小確認

```bash
cd webcam-bridge
python3 webcam_litter_bridge.py \
  --mode mock \
  --camera-id 401 \
  --output-dir ../mediator-owner/raw_data/output \
  --flush-seconds 5
```

mock でパイプラインの動作を確認できたら、次は USB ウェブカメラ実機で検知を試します。

## Phase 1: 実機 webcam モードで検知を確認する

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
