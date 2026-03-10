# HUSKYLENS2サンプル

## 目的

HUSKYLENS2（または中継入力）からイベントを作り、商品化までを確認します。

## このページで分かること

- HUSKYLENS2 の検知結果がどのようにイベントファイルへ変わるか
- `mock` と `serial` のどちらで試すべきか
- 商品化まで進んだときに、どの出力を見ればよいか

## つまずきやすい点

- シリアルポート名や権限の確認で止まりやすい
- `sensor-bridge` の出力先と `mediator-owner` の監視先がずれると先に進まない
- 実機でうまくいかないときは、まず `mock` でパイプラインだけ確認する方が切り分けやすい

## 公式リンク

- HUSKYLENS2 製品ページ: <https://www.dfrobot.com/product-2995.html>
- DFRobot 公式サイト: <https://www.dfrobot.com/>

## 前提

- HUSKYLENS2 本体、または mock モードでの確認環境がある
- `sensor-bridge` が利用できる
- `mediator-owner` が起動済みで、`raw_data/output` を監視している
- シリアル接続時は、PCからデバイスのポート名が見えている

## 演習用プログラム

- [問題用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/huskylens2_mock/problem_program.py)
- [解答用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/huskylens2_mock/answer_program.py)
- [演習説明](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/huskylens2_mock/README.md)

この演習では、`mediator-owner/raw_data/output` に出力する mock イベントファイルを自分で組み立てます。  
問題用プログラムでは `build_event()` を完成させ、HUSKYLENS2 検知イベントの最小 JSON を理解するのが目的です。

## 最短ルート

最初は次の 4 手順で十分です。

1. `mock` モードで `huskylens_bridge.py` を起動する
2. `raw_data/output` にイベントファイルが出ることを確認する
3. `mediator-owner` がそのファイルを拾うことを確認する
4. フロントに商品が反映されることを確認する

その後の分岐:

- まずパイプラインだけ確認したい場合: `mock` モードだけで十分です
- 実機接続まで確認したい場合: 次に `serial` モードを試します
- シリアルで止まる場合: 先にトラブル項目を見た方が早いです

## 工程別の目次

<details class="iw3ip-toc-details" open>
  <summary>確認 1: mock モードでイベント生成を確認する</summary>
  <p>最初に mock モードでイベントファイル生成を確認し、センサ入力がなくても後段のパイプラインが動くことを確かめます。</p>
  <ol>
    <li><a href="#1-mockで最小確認">mockで最小確認</a></li>
    <li><a href="#3-確認ポイント">確認ポイント</a></li>
    <li><a href="#成功例">成功例</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 2: 実機 serial モードで同じ流れを試す</summary>
  <p>次に HUSKYLENS2 実機を使って同じ出力先へイベントを出し、mock と同じ後段が利用できることを確認します。</p>
  <ol>
    <li><a href="#2-実機serial">実機（serial）</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 3: トラブル時の切り分けを行う</summary>
  <p>最後に、シリアルポート、依存ライブラリ、出力先パスのような典型的な失敗点を切り分けます。</p>
  <ol>
    <li><a href="#4-トラブル時">トラブル時</a></li>
  </ol>
</details>

## 読み進め方

このページは Phase 1 のデバイス接続ページなので、最初から実機で進める必要はありません。むしろ、まず `mock` で後段パイプラインを確認し、その後で `serial` に切り替える方が問題の切り分けがしやすくなります。

## Phase 1: mock モードでパイプラインを確認する

## 1. mockで最小確認

```bash
cd sensor-bridge
python3 huskylens_bridge.py \
  --mode mock \
  --camera-id 301 \
  --output-dir ../mediator-owner/raw_data/output \
  --flush-interval-sec 8
```

ここまでで、後段のパイプライン自体は確認できています。次は同じ流れを実機 serial 入力で試します。

## Phase 1: 実機 serial モードで同じ流れを試す

## 2. 実機（serial）

```bash
python3 huskylens_bridge.py \
  --mode serial \
  --serial-port /dev/ttyUSB0 \
  --baudrate 115200 \
  --camera-id 301 \
  --output-dir ../mediator-owner/raw_data/output
```

## 3. 確認ポイント

- `raw_data/output` に `301_huskylens_*.txt` が作成される
- `mediator-owner` ログに watcher event が出る
- フロントに商品が反映される

## 成功例

- mock モードでもイベントファイルが生成される
- 実機モードでは、検出対象に応じて継続的にイベントが出力される
- フロントエンドに商品が現れ、購入操作まで進める

## 4. トラブル時

- 症状: `pyserial` が見つからない
  - 対応: `pip install pyserial`
- 症状: シリアルポート名が分からない
  - 確認: `/dev/tty.usbserial-*` などのデバイス名を確認
- 症状: デバイスが読めない
  - 確認: ケーブルが給電専用でなく通信対応か
  - 確認: macOS / Linux の権限やポート名が正しいか
- 症状: イベントが商品化されない
  - 確認: `mediator-owner` が起動しているか
  - 確認: 出力先が `../mediator-owner/raw_data/output` と一致しているか
