# AI カメラで人物を検知する (HUSKYLENS2)

HUSKYLENS2 という小型 AI カメラからイベントを取り出し、マーケットに出品するまでの一連を試します。

> **やること**: HUSKYLENS2 (または mock) の検知結果からイベントファイルを生成する
>
> **前提**: [環境構築](../setup/index.md) と [最短起動](../setup/quickstart.md)
>
> **使うもの**: PC + HUSKYLENS2 (なければ [USB ウェブカメラサンプル](webcam.md) で代替)
>
> **所要時間**: 45 分くらい

## 目的

HUSKYLENS2（または mock の入力）の検知結果を数秒ごとにまとめ、イベントファイルとして出力します。

!!! warning "商品として登録されるところまでは、現在は確認できません"
    現在の `main` の `mediator-owner` が商品として登録するのは、起動時に `raw_data/output` にある動画 (`<カメラ ID>_movie_<番号>.mp4`) と同名の `.json` の組だけです。このページのブリッジが出力する `.txt` のイベントファイルは、商品として登録されません。このページで確認できるのは、イベントファイルの生成までです。


## このページで分かること

- HUSKYLENS2 の検知結果がどのようにイベントファイルへ変わるか
- `mock` と `serial` のどちらで試すべきか
- 生成されたイベントファイルのどこを見ればよいか

## つまずきやすい点

- シリアルポート名や権限の確認で止まりやすい
- 出力先のフォルダが無いと、イベントファイルを書き出せない
- 実機でうまくいかないときは、まず `mock` でパイプラインだけ確認する方が切り分けやすい

## 公式リンク

- HUSKYLENS2 製品ページ: <https://www.dfrobot.com/product-2995.html>
- DFRobot 公式サイト: <https://www.dfrobot.com/>

## 前提

- HUSKYLENS2 本体、または mock モードでの確認環境がある
- `sensor-bridge` が利用できる
- 出力先のフォルダ `mediator-owner/raw_data/output` がある (無ければ手順の中で作ります)
- シリアル接続時は、PCからデバイスのポート名が見えている
- Python 3 が使える（serial モードでは `pyserial` も必要）

## 演習用プログラム

- [問題用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/huskylens2_mock/problem_program.py)
- [解答用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/huskylens2_mock/answer_program.py)
- [演習説明](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/huskylens2_mock/README.md)

この演習では、`mediator-owner/raw_data/output` に出力する mock イベントファイルを自分で組み立てます。  
問題用プログラムでは `build_event()` を完成させ、HUSKYLENS2 検知イベントの最小 JSON を理解するのが目的です。

## 最短ルート

最初は次の 3 手順で十分です。

1. `mock` モードで `huskylens_bridge.py` を起動する
2. `raw_data/output` にイベントファイルが出ることを確認する
3. ファイルの中身 (検知した対象と件数) を確認する

その後の分岐:

- まずパイプラインだけ確認したい場合: `mock` モードだけで十分です
- 実機接続まで確認したい場合: 次に `serial` モードを試します
- シリアルで止まる場合: 先に [トラブル時](#4-トラブル時) を確認してください

## Phase 1: mock モードでパイプラインを確認する

### 1. mockで最小確認

教材リポジトリのトップ (`Blockchain_IoT_Marketplace/`) から実行します。mock モードは、機器の代わりに疑似的な検知結果を生成します。

```bash
cd sensor-bridge
mkdir -p ../mediator-owner/raw_data/output
python3 huskylens_bridge.py \
  --mode mock \
  --camera-id 301 \
  --output-dir ../mediator-owner/raw_data/output \
  --flush-interval-sec 8
```

| 引数 | 意味 |
|---|---|
| `--mode` | 入力の種類。`mock` (疑似データ)、`serial` (シリアル接続の実機)、`tcp` |
| `--camera-id` | ファイル名の先頭に付くカメラ ID |
| `--output-dir` | イベントファイルの出力先 |
| `--flush-interval-sec` | 何秒ごとに検知結果をまとめて 1 ファイルにするか |

8 秒ごとに 1 行ずつ、次のように表示されます。止めるときは `Ctrl+C` を押します。

```
[bridge] starting HuskyLens2 bridge
[bridge] mode=mock camera_id=301 output_dir=../mediator-owner/raw_data/output
[bridge] flush_interval_sec=8 min_confidence=0.50
[bridge] batch detections=12 file=../mediator-owner/raw_data/output/301_huskylens_1791549165.txt
[bridge] batch detections=13 file=../mediator-owner/raw_data/output/301_huskylens_1791549173.txt
```

出力されたファイルの中身を確認します。

```bash
ls ../mediator-owner/raw_data/output
cat ../mediator-owner/raw_data/output/301_huskylens_*.txt | head -20
```

```
# HuskyLens2 Detection Snapshot
camera_id: 301
window_start_utc: 2026-10-09T12:32:37.144510+00:00
window_end_utc: 2026-10-09T12:32:45.178510+00:00
total_detections: 12
unique_labels: 3
top_label: bike
max_confidence: 0.956
labels:
  - bike: 5
  - person: 4
  - car: 3
samples:
  - {"timestamp": "2026-10-09T12:32:38.149827+00:00", "label": "bike", "confidence": 0.6867, "id": "17", "x": 50.54, "y": 461.24, "w": 28.13, "h": 180.54, "source": "mock"}
  ...
```

1 ファイルが、8 秒間の検知結果をまとめた 1 件のイベントです。`labels` は検知した対象ごとの件数、`samples` は個々の検知 (時刻、ラベル、信頼度、画面上の位置と大きさ) です。

mock でイベントファイルが生成されることを確認できたら、次は実機の serial 入力を試します。

## Phase 1: 実機 serial モードで同じ流れを試す

### 2. 実機（serial）

```bash
python3 huskylens_bridge.py \
  --mode serial \
  --serial-port /dev/ttyUSB0 \
  --baudrate 115200 \
  --camera-id 301 \
  --output-dir ../mediator-owner/raw_data/output
```

### 3. 確認ポイント

- `raw_data/output` に `301_huskylens_*.txt` が作成される
- ファイルの `labels` に、検知した対象 (person など) と件数が入っている

## 成功例

- mock モードでもイベントファイルが生成される
- 実機モードでは、検出対象に応じて継続的にイベントが出力される

### 4. トラブル時

- 症状: `pyserial` が見つからない
  - 対応: `pip install pyserial`
- 症状: シリアルポート名が分からない
  - 確認: `/dev/tty.usbserial-*` などのデバイス名を確認
- 症状: デバイスが読めない
  - 確認: ケーブルが給電専用でなく通信対応か
  - 確認: macOS / Linux の権限やポート名が正しいか
- 症状: イベントファイルが作成されない
  - 確認: 出力先のフォルダ (`../mediator-owner/raw_data/output`) が存在するか。無ければ `mkdir -p ../mediator-owner/raw_data/output` で作る
- 症状: イベントが商品として登録されない
  - ページ冒頭の注意のとおり、現在の `mediator-owner` は `.txt` のイベントファイルを登録しません
