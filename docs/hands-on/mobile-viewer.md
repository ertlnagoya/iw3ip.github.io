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

## 最短ルート

最初は次の 4 手順で十分です。

1. `iot-market-ui` を `0.0.0.0:5173` で起動する
2. PC の LAN IP を確認する
3. スマホで `http://<LAN_IP>:5173/mobile` を開く
4. イベント一覧が見えることを確認する

その後の分岐:

- 閲覧だけ確認したい場合: `/mobile` が開いて一覧が見えれば十分です
- 購入まで確認したい場合: MetaMask モバイルのブラウザで開きます
- 接続で止まる場合: 最後のトラブル項目を先に確認した方が早いです

## 工程別の目次

<details class="iw3ip-toc-details" open>
  <summary>準備: PC 側の公開設定と LAN IP を確認する</summary>
  <p>最初に PC 側のフロントエンドを LAN 公開で起動し、スマホからアクセスするための IP アドレスを確認します。</p>
  <ol>
    <li><a href="#1-pc側をlan公開で起動">PC側をLAN公開で起動</a></li>
    <li><a href="#2-pcのlan-ipを確認">PCのLAN IPを確認</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 1: スマホから閲覧画面を開く</summary>
  <p>次に、スマホから `/mobile` を開き、一覧画面とフィルタが見えることを確認します。</p>
  <ol>
    <li><a href="#3-スマホでアクセス">スマホでアクセス</a></li>
    <li><a href="#5-確認ポイント">確認ポイント</a></li>
    <li><a href="#成功例">成功例</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 2: 購入操作と接続トラブルを確認する</summary>
  <p>最後に、必要ならウォレット経由の購入操作を試し、つながらない場合の代表的な確認ポイントを整理します。</p>
  <ol>
    <li><a href="#4-購入時の注意">購入時の注意</a></li>
    <li><a href="#トラブル時">トラブル時</a></li>
  </ol>
</details>

## 読み進め方

Phase 1 で共有された結果をスマホから確認する手順です。短時間なら URL を開いて一覧を見るだけでも OK。ハンズオンとしては「同じ LAN 上で動く」「ウォレット経由の購入入口」まで確認するのが望ましい。

## 接続の前提を整える

## 1. PC側をLAN公開で起動

```bash
cd iot-market-ui
npm run dev -- --host 0.0.0.0 --port 5173
```

## 2. PCのLAN IPを確認

```bash
ipconfig getifaddr en0
```

スマホからアクセスする URL を組み立てられます。実際にスマホでページを開いて確認しましょう。

## スマホで閲覧画面を確認する

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

閲覧用 UI への接続確認は完了です。必要なら購入操作を試し、接続できない場合は以下のトラブル項目を確認してください。

## 購入操作とトラブル確認

## トラブル時

- 症状: スマホからアクセスできない
  - 確認: PC とスマホが同じネットワークか
  - 確認: PC 側のファイアウォールで遮断されていないか
  - 確認: PC の LAN IP が変わっていないか
- 症状: 購入操作が進まない
  - 確認: MetaMask モバイルの in-app browser を使っているか
  - 確認: ローカルチェーン設定が MetaMask に入っているか
