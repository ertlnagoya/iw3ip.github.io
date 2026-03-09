# スマホ閲覧アプリ

## 目的

スマホからイベントフィードを閲覧し、必要なデータを購入できることを確認します。

## このページで分かること

- スマホから閲覧用 UI に接続するために何が必要か
- `http://<LAN_IP>:5173/mobile` の URL が何を意味しているか
- 閲覧確認と購入確認で見るべき画面

## つまずきやすい点

- PC 側を `0.0.0.0` で公開していないとスマホから届かない
- 同じ Wi-Fi に見えてもネットワークが分かれていると接続できない
- 購入操作は MetaMask モバイルのブラウザ経由でないと進みにくい

## 公式リンク

- MetaMask Mobile: <https://metamask.io/>
- Vite（`npm run dev` の開発サーバで使用）: <https://vite.dev/>

## 前提

- `iot-market-ui` が起動できる
- PC とスマホが同じ LAN / Wi-Fi に接続されている
- スマホで MetaMask モバイルアプリ、または同等のウォレットブラウザが使える

## 演習用プログラム

- [問題用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/mobile_viewer/problem_program.py)
- [解答用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/mobile_viewer/answer_program.py)
- [演習説明](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/mobile_viewer/README.md)

この演習では、PC の LAN IP からスマホ用 URL を組み立てます。  
問題用プログラムでは `http://<LAN_IP>:5173/mobile` を組み立てる関数を完成させ、接続先 URL の意味を確認します。

## 1. PC側をLAN公開で起動

```bash
cd iot-market-ui
npm run dev -- --host 0.0.0.0 --port 5173
```

## 2. PCのLAN IPを確認

```bash
ipconfig getifaddr en0
```

## 3. スマホでアクセス

```txt
http://<PCのLAN_IP>:5173/mobile
```

例:

```txt
http://192.168.1.20:5173/mobile
```

## 4. 購入時の注意

- スマホで購入する場合は MetaMaskモバイル内ブラウザを推奨

## 5. 確認ポイント

- イベントフィードが表示される
- フィルタが使える
- 購入後、履歴タブに記録される

## 成功例

- スマホから `/mobile` 画面が開く
- 商品一覧やイベント一覧が表示される
- MetaMask モバイル経由で購入確認画面に進める

## トラブル時

- 症状: スマホからアクセスできない
  - 確認: PC とスマホが同じネットワークか
  - 確認: PC 側のファイアウォールで遮断されていないか
  - 確認: PC の LAN IP が変わっていないか
- 症状: 購入操作が進まない
  - 確認: MetaMask モバイルの in-app browser を使っているか
  - 確認: ローカルチェーン設定が MetaMask に入っているか
