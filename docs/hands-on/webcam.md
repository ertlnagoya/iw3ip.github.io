# USB カメラでポイ捨てを検知する (OpenCV)

HUSKYLENS2 がなくても、USB ウェブカメラと OpenCV だけで `person_detected` / `possible_littering` イベントを作れます。

> **やること**: USB カメラ画像から人物・ポイ捨て検知をしてイベントファイルを生成
>
> **前提**: [環境構築](../setup/index.md) と [最短起動](../setup/quickstart.md)
>
> **使うもの**: PC + USB ウェブカメラ
>
> **所要時間**: 30 分くらい

## 目的

USB ウェブカメラの画像から人物とポイ捨ての可能性を検知し、イベントファイルを生成します。

!!! warning "商品として登録されるところまでは、現在は確認できません"
    現在の `main` の `mediator-owner` が商品として登録するのは、起動時に `raw_data/output` にある動画 (`<カメラ ID>_movie_<番号>.mp4`) と同名の `.json` の組だけです。このページのブリッジが出力する `.txt` のイベントファイルは、商品として登録されません。このページで確認できるのは、イベントファイルの生成までです。


## このページで分かること

- USBウェブカメラだけで Phase 1 のイベント生成を試す方法
- `person_detected` と `possible_littering` をどのように使い分けるか
- 生成されたイベントファイルのどこを見ればよいか

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
- 出力先のフォルダ `mediator-owner/raw_data/output` がある (無ければ手順の中で作ります)
- 実機モードでは、PCからカメラデバイスが認識されている
- Python 3 が使える（webcam モードでは `pip install ultralytics opencv-python` も必要）

## 演習用プログラム

- [問題用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/webcam_littering_mock/problem_program.py)
- [解答用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/webcam_littering_mock/answer_program.py)
- [演習説明](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/webcam_littering_mock/README.md)

この演習では、USBウェブカメラの mock 出力として `possible_littering` イベントファイルを生成します。  
問題用プログラムでは、イベントの最小構造と、`camera_id`・`confidence`・`event_type` の意味を確認できます。

## 最短ルート

最初は次の 3 手順で十分です。

1. `mock` モードで `webcam_litter_bridge.py` を起動する
2. `*_webcam_event_*.txt` が生成されることを確認する
3. ファイルの中身 (イベントの種類と確からしさ) を確認する

その後の分岐:

- まずパイプラインだけ確認したい場合: `mock` モードで十分です
- 実機のカメラ検知まで見たい場合: `webcam` モードへ進みます
- カメラが開けない場合: `camera-index` と占有状況の確認を先に行います

## Phase 1: mock モードでパイプラインを確認する

### 1. mockで最小確認

教材リポジトリのトップ (`Blockchain_IoT_Marketplace/`) から実行します。mock モードは、カメラの代わりに疑似的なイベントを生成します (追加のパッケージは不要です)。

```bash
cd webcam-bridge
mkdir -p ../mediator-owner/raw_data/output
python3 webcam_litter_bridge.py \
  --mode mock \
  --camera-id 401 \
  --output-dir ../mediator-owner/raw_data/output \
  --flush-seconds 5
```

5 秒ごとに 1 行ずつ、次のように表示されます。止めるときは `Ctrl+C` を押します。

```
[webcam-bridge] mock mode start
[webcam-bridge] wrote ../mediator-owner/raw_data/output/401_webcam_event_1791549330.txt
[webcam-bridge] wrote ../mediator-owner/raw_data/output/401_webcam_event_1791549335.txt
```

出力されたファイルの中身を確認します。

```bash
cat ../mediator-owner/raw_data/output/401_webcam_event_*.txt | head -10
```

```
# Webcam Event Snapshot
camera_id: 401
event_type: possible_littering
event_time_utc: 2026-10-09T12:35:30.906857+00:00
event_score: 0.801
summary: Bottle remained on ground area for long duration
details:
{"event_type": "possible_littering", "event_time_utc": "...", "event_score": 0.801, "summary": "Bottle remained on ground area for long duration", "object": {...}}
```

1 ファイルが 1 件のイベントです。`event_type` がイベントの種類、`event_score` が確からしさ、`details` が元になった検知の内容です。

mock でイベントファイルが生成されることを確認できたら、次は USB ウェブカメラ実機で検知を試します。

## Phase 1: 実機 webcam モードで検知を確認する

### 2. 実機（USB webcam）

実機モードでは、物体検出のモデル (YOLO) を使います。`pip install ultralytics opencv-python` で必要なパッケージを入れてください。初回の実行時に、モデルのファイル (`yolov8n.pt`、数 MB) が自動でダウンロードされます。

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

### 3. 判定ロジック（簡易）

- 人を検出 → `person_detected`
- `bottle/cup` が一定時間残留し、人が近くにいない → `possible_littering`

### 4. 確認ポイント

- `*_webcam_event_*.txt` が生成される
- ファイルの `event_type` が `person_detected` または `possible_littering` になっている

## 成功例

- mock モードでイベントファイル生成まで確認できる
- 実機モードで `person_detected` または `possible_littering` が出力される

### 5. 注意

この検知は、学習用の簡易な規則（ヒューリスティック）によるもので、厳密な判定ではありません。

## トラブル時

- 症状: カメラが開けない
  - 確認: 別アプリがカメラを占有していないか
  - 確認: `--camera-index 0` を `1` などに変えて試したか
- 症状: 期待イベントが出ない
  - 確認: 照明条件や画角が適切か
  - 確認: mock モードでパイプライン自体が動くか
- 症状: イベントファイルが作成されない
  - 確認: 出力先のフォルダ (`../mediator-owner/raw_data/output`) が存在するか。無ければ `mkdir -p ../mediator-owner/raw_data/output` で作る
- 症状: イベントが商品として登録されない
  - ページ冒頭の注意のとおり、現在の `mediator-owner` は `.txt` のイベントファイルを登録しません
