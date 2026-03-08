# HUSKYLENS2サンプル

## 目的

HUSKYLENS2（または中継入力）からイベントを作り、商品化までを確認します。

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

## 1. mockで最小確認

```bash
cd sensor-bridge
python3 huskylens_bridge.py \
  --mode mock \
  --camera-id 301 \
  --output-dir ../mediator-owner/raw_data/output \
  --flush-interval-sec 8
```

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
