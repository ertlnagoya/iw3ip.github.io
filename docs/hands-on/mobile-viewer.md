# スマホでマーケットを開いて購入する (v1)

v1 マーケットプレイス (MetaMask で購入し、データは simple-storage (:3000) 経由で受け渡す) を、スマホのブラウザで開いて購入するところまで確かめます。VC 連携 (v2) の動線は別ページです。

> **やること**: スマホのブラウザでマーケットプレイス画面を開き、購入操作まで確かめる
>
> **前提**: [最短起動](../setup/quickstart.md) で v1 スタックが立ち上がっていること
>
> **使うもの**: PC + スマホ (同じネットワーク)
>
> **所要時間**: 約 20 分

!!! note "v1 で開いて購入する手順"
    このページは **v1 (現行マーケットプレイス + MetaMask)** で、
    スマホからマーケットを開いて購入する手順です。VC 連携 (v2) のスマホ動線は
    [スマホアプリで購入から閲覧まで (Stage A)](marketplace-mobile-app.md) を
    参照してください。

## 目的

スマホからマーケットを開き、商品を購入できることを確かめます。

## このページで分かること

- スマホからマーケット画面に接続するために何が必要か
- iot-market-ui が公開するルート (`/`, `/merchandise/[address]`,
  `/purchased/[txHash]`, `/seller` 等) の役割
- 開いたとき・購入したときに見るべき画面

!!! info "`/mobile` は実装されていません"
    過去の演習用プログラム (`examples/hands_on/mobile_viewer/`) は
    架空の `/mobile` ルートを前提にしていますが、現在の `iot-market-ui`
    にこのルートはありません。実際に開く URL は「最短ルート」で示します。

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

- [問題用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/mobile_viewer/problem_program.py)
- [解答用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/mobile_viewer/answer_program.py)
- [演習説明](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/mobile_viewer/README.md)

この演習では、PC の LAN IP からスマホ用 URL を組み立てます。
問題用プログラムは架空の `/mobile` 前提なので、現在の UI で動かす場合は
homepage `/` をスマホで開く形に読み替えてください。

## 最短ルート

最初は次の 5 手順で十分です。

1. `iot-market-ui/.env.local` を用意し、接続先を自分の環境に合わせる (PC のみなら `127.0.0.1`、スマホからなら LAN IP)
2. `iot-market-ui` を `0.0.0.0:5173` で起動する
3. PC の LAN IP を確認する
4. スマホで `http://<HOST_IP>:5173/` を開く (homepage)
5. 検索 UI が見えることを確認する

その後の分岐:

- **個別の商品を見る場合**: `http://<HOST_IP>:5173/merchandise/<address>` を直接開く
  (Hardhat デプロイ後の Merchandise アドレスを指定)
- **購入まで確認したい場合**: MetaMask モバイルのブラウザで開きます
- **VC 連携 (v2) を試したい場合**: [スマホアプリで購入から閲覧まで (Stage A)](marketplace-mobile-app.md)
  に進みます
- 接続で止まる場合: 先に [トラブル時](#トラブル時) を確認してください

## 接続の前提を整える

### 0. フロントの接続先を設定 (.env.local) { #env-local }

`iot-market-ui` は、ブロックチェーン (RPC) と publisher の接続先を `iot-market-ui/.env.local` から読み込みます。接続先が間違っていると、商品ページが **HTTP 500** になります (ファイルが無い場合は RPC に `127.0.0.1`、publisher に `localhost` が使われるため、スマホからは接続できません)。まず雛形をコピーして自分の環境に合わせます。

```bash
cd iot-market-ui
cp .env.example .env.local
```

`.env.local` を編集し、利用シーンに合わせて host を設定します。

- **PC だけで確認する場合**: `127.0.0.1` のままで構いません。

    ```
    VITE_RPC_URL=http://127.0.0.1:8545
    VITE_PUBLISHER_URL=http://127.0.0.1:8080
    ```

- **スマホから操作する場合**: PC の LAN IP (手順2で確認) に置き換えます。

    ```
    VITE_RPC_URL=http://<HOST_IP>:8545
    VITE_PUBLISHER_URL=http://<HOST_IP>:8080
    ```

!!! warning "スマホから使うときは Hardhat ノードも LAN 公開する"
    既定の `npx hardhat node` は `127.0.0.1` だけで待ち受けるため、スマホや
    MetaMask モバイルから届きません。スマホで操作する場合は、ノードを
    `npx hardhat node --hostname 0.0.0.0` で起動し直し、PC のファイアウォールで
    `8545` を許可してください。

    ノードを起動し直すとチェーンが空に戻り、デプロイしたコントラクトは消えます。
    起動し直したら [最短起動](../setup/quickstart.md) の手順 2 (デプロイ) をやり直し、
    `mediator-owner` と `mediator-buyer` も再起動してください。以前の商品アドレスは使えません。

`.env.local` を変更したら、フロント (`npm run dev`) を再起動して反映します。

### 1. PC側をLAN公開で起動

手順 0 で `iot-market-ui` に移動したままのターミナルで実行します。

```bash
npm run dev -- --host 0.0.0.0 --port 5173
```

### 2. PCのLAN IPを確認

```bash
ipconfig getifaddr en0
```

`ipconfig getifaddr en0` は macOS のコマンドです。Windows では `ipconfig`、Linux では `ip addr` で確認します。以下の手順に出てくる `<HOST_IP>` は、ここで得られた IP アドレスに置き換えてください (OS 別の調べ方は [ハンズオンの概要](index.md#host-ip) にもまとめています)。

## スマホでマーケットを開く

### 3. スマホでアクセス

```txt
http://<HOST_IP>:5173/
```

例 (homepage):

```txt
http://192.168.1.20:5173/
```

個別の商品を直接開く場合 (Hardhat にデプロイされた Merchandise アドレスを指定):

```txt
http://<HOST_IP>:5173/merchandise/<merchandise_address>
```

### 4. 購入時の注意

- スマホで購入する場合は MetaMaskモバイル内ブラウザを推奨
- MetaMask モバイルにローカルチェーンのネットワークを追加しておく:
    - RPC URL: `http://<HOST_IP>:8545` (例: `http://192.168.1.20:8545`)
    - チェーン ID: `31337` / 通貨記号: `ETH`
- MetaMask モバイルにも **Account #2 の Private Key** をインポートし、そのアカウントで購入する ([最短起動](../setup/quickstart.md#metamask-のローカルチェーン設定) と同じ鍵)。公開鍵が登録されているのは、仲介プロセスが使う Account #1 (`mediator-owner`) と Account #2 (`mediator-buyer`) だけです。購入者側の仲介プロセスは Account #2 として動くため、ほかのアカウントで購入すると、取引が失敗 (revert) するか、データが届きません
- ネットワーク追加が拒否される場合は、ノードを `--hostname 0.0.0.0` で起動しているか、`.env.local` と MetaMask の RPC が **127.0.0.1 ではなく PC の LAN IP** になっているかを確認

### 5. 確認ポイント

- iot-market-ui の検索 UI または商品ページがスマホで開ける
- 個別 Merchandise ページで購入操作が走る
- 購入後は `/purchased/[txHash]` に移ります。この画面は publisher の `/marketplace/claim` を呼んで VC 受け取り用の QR を出す v2 用の画面です。v1 では publisher を起動していないため「エラー: ...」と表示されますが、購入自体は完了しています
- 購入できたかは PC の `mediator-buyer` のログで確かめます。`You bought a product of ...` に続いて `File downloaded successfully` と `Verification result: true` が出れば、データが simple-storage (:3000) から届いています

## 成功例

- スマホから iot-market-ui ホームが開く
- 商品ページから購入動線まで進める
- MetaMask モバイル経由で購入確認画面に進める

- `mediator-buyer` のログに購入とダウンロードの行が出る

接続できない場合や購入が進まない場合は、以下のトラブル項目を確認してください。

## トラブル時

- 症状: スマホからアクセスできない
  - 確認: PC とスマホが同じネットワークか
  - 確認: PC 側のファイアウォールで遮断されていないか
  - 確認: PC の LAN IP が変わっていないか
- 症状: 商品ページが HTTP 500 になる
  - 確認: `iot-market-ui/.env.local` の `VITE_RPC_URL` が、起動中の Hardhat ノード (`8545`) を指しているか
  - 確認: 接続先が他人の IP や `8546` など誤った値になっていないか
  - 詳しくは [トラブルシュート](../operations/troubleshooting.md) の「商品ページが 500」項目を参照
- 症状: 購入操作が進まない / MetaMask にネットワークを追加できない
  - 確認: MetaMask モバイルの in-app browser を使っているか
  - 確認: MetaMask の RPC とノードの公開設定が一致しているか (`--hostname 0.0.0.0` + LAN IP)
  - 詳しくは [トラブルシュート](../operations/troubleshooting.md) の「MetaMask にネットワークを追加できない」項目を参照
- 症状: 購入の取引が失敗する (revert) / `mediator-buyer` に `You bought a product of` が出ない
  - 確認: MetaMask モバイルで選んでいるアカウントが Account #2 か (別のアカウントは公開鍵が登録されていないため購入できない)
  - 対処: [最短起動](../setup/quickstart.md#metamask-のローカルチェーン設定) と同じ Account #2 の Private Key をインポートし、そのアカウントで購入し直す
  - 詳しくは [トラブルシュート](../operations/troubleshooting.md) の「購入できない」項目を参照
