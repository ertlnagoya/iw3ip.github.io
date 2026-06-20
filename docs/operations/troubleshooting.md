# トラブルシュート

よくある症状ごとに確認点と対応をまとめています。症状に近い項目を選び、上から順に確認してください。

## 1. `host.docker.internal` の接続エラー

- 症状: 仲介プロセス (`mediator-owner` / `mediator-buyer`) やフロントが `host.docker.internal` に接続できず、DNS 解決に失敗する
  - ログ例: `ConnectError("dns error", ... "failed to lookup address information: nodename nor servname provided, or not known")`
  - フロント例: `JsonRpcProvider failed to detect network ... (perhaps the URL is wrong or the node is not started)` が 1 秒おきに出続ける
  - 原因: `host.docker.internal` は Docker Desktop が **コンテナ内にのみ** 提供する特別なホスト名です。`cargo run` や `npm run dev` で **コンテナの外 (ホスト OS でネイティブ実行)** している場合、この名前は解決できません。

### 確認

- ノード自体が生きているかは、ホスト側で次を実行して確認します。

  ```bash
  curl http://localhost:8545
  ```

  `{"jsonrpc":"2.0", ... "error": ... "Parse error: Unexpected end of JSON input"}` が返れば、**ノードは到達可能で正常** です (空ボディの GET に対する JSON-RPC の正常な応答であり、エラーではありません)。

### 対処 A — ネイティブ実行のまま直す (quickstart の `cargo run` 手順)

設定ファイル中の `host.docker.internal` を `localhost` に読み替えます。

- `mediator-owner/settings/owner_1.yaml`:

  ```yaml
  api_url: "http://localhost:3000"
  rpc_url: "http://localhost:8545"
  ```

- 購入者側 `mediator-buyer` の対応する設定、およびフロント (`iot-market-ui`) の RPC 接続先設定にも同様に適用します。接続先が見当たらない場合は、リポジトリ内を `host.docker.internal` で grep して洗い出してください。
- 変更後、各プロセスを再起動します。

### 対処 B — コンテナ内で動かす

仲介プロセスやフロントを DevContainer / Docker 側で実行すれば、`host.docker.internal` がそのまま解決します。設定を変更したくない場合はこちらを選びます。

## 2. IPFSコンテナが再起動ループ

- 症状: IPFS コンテナが再起動を繰り返す
  - 確認: `docker compose ps` で IPFS だけが継続的に再起動していないか
  - 対応: `ipfs/docker-compose.yaml` の image を `ipfs/kubo:latest` にする

## 3. スマホから `/mobile` が開けない

- 症状: スマホから `/mobile` が開けない
  - 確認: `npm run dev -- --host 0.0.0.0 --port 5173` で起動したか
  - 確認: PC とスマホが同一ネットワークか
  - 確認: PC のファイアウォールで `5173` が遮断されていないか

## 4. 商品が表示されない

- 症状: 商品が表示されない
  - 確認: `mediator-owner` の実行引数が `cargo run -- settings/owner_1.yaml` になっているか
  - 確認: `raw_data/output` にファイルが作成されているか
  - 対応: センサ入力側から商品生成までの手順を再実行し、途中の出力ファイル作成を確認する

## 5. 購入できない

- 症状: 購入できない
  - 確認: MetaMask の接続アカウントが正しいか
  - 対応: ローカルチェーン再起動後は MetaMask のキャッシュをクリア

## 6. イベントが共有されず `no_matching_consent` で拒否される

- 症状: HA Demo Simulator などで、一部のデータセットだけ `/platform/ingest` に出てこない。`audit/logs` を見ると `action: deny` / `reason: no_matching_consent` / `subject_did: "unknown"` になっている
  - ログ例:

    ```json
    {"action":"deny","subject_did":"unknown","dataset_id":"home/event/possible_littering","purpose":"research","reason":"no_matching_consent", ...}
    ```

  - 原因の第一候補: **consent VC の有効期限切れ**。consent ファイル (`examples/ha_demo/consent_*.json`) の `valid_to` が過去日付だと、登録しても現在時刻が有効期間外となり、一致せず拒否されます。教材に含まれるサンプルは固定日付のため、時間が経つと期限切れになります。

### 確認

- まず audit ログを読みます (zsh では `?` がワイルドカード扱いになるため URL をクォートで囲みます)。

  ```bash
  curl 'http://localhost:8080/audit/logs?limit=20'
  ```

- 拒否されている `dataset_id` の consent ファイルを開き、`valid_from` 〜 `valid_to` の範囲に**今日の日付**が入っているか確認します。`valid_to` が過去なら期限切れです。

### 対処

- consent ファイルの `valid_to` を十分先の日付に延ばし、登録し直します。期限切れの行をまとめて延長する例 (macOS の `sed` は `-i ''` が必要):

  ```bash
  sed -i '' 's/"valid_to": "2026-05-01T00:00:00Z"/"valid_to": "2027-12-31T23:59:59Z"/' examples/ha_demo/consent_*.json
  ```

- 編集後、consent を登録し直し (各レスポンスが `"status":"stored"` になることを確認)、イベントを送り直してから `/platform/ingest` を再確認します。
- 既存の `deny` ログは履歴として残ります。必ず **consent 登録後に送り直した新しいイベント** で結果を確認してください。
- 恒久対応: 教材リポジトリ側で consent の `valid_to` を長期 (または生成時に「現在 + 1 年」など) に修正します。

## 7. 商品ページが 500 になる (iot-market-ui の接続先)

- 症状: 一覧 (`/`) は開けるのに、個別の商品ページ `http://localhost:5173/merchandise/<address>` を開くと **HTTP 500** になる
  - 原因: フロント (`iot-market-ui`) は RPC と publisher の接続先を `iot-market-ui/.env.local` から読み込みます。この接続先が **起動中の Hardhat ノードに届かない値** (他人の IP、`8546` などの誤ポート、未起動のホスト) だと、商品ページのデータ取得が失敗し 500 になります。一覧ページはタイムアウト処理があるため描画はされますが、商品詳細ページは失敗がそのまま 500 として表面化します。

### 確認・対処

- `iot-market-ui/.env.local` を開き、`VITE_RPC_URL` が起動中のノード (既定は `http://127.0.0.1:8545`) を指しているか確認します。ファイルが無ければ雛形からコピーします。

  ```bash
  cd iot-market-ui
  cp .env.example .env.local
  ```

  ```
  VITE_RPC_URL=http://127.0.0.1:8545
  VITE_PUBLISHER_URL=http://127.0.0.1:8080
  ```

- スマホから操作する場合は `127.0.0.1` を PC の LAN IP に置き換えます (例: `http://192.168.1.20:8545`)。
- 変更後はフロント (`npm run dev`) を再起動します。
- 補足: コントラクトアドレス不一致が疑われる場合は、Hardhat ノードを一度止めて起動し直し (`npx hardhat node`)、再デプロイすると既定アドレスに揃います。それでもズレる場合は `iot-market-ui/src/lib/contracts/IoTMarket.ts` と `PubKey.ts` を実デプロイのアドレスに更新します。

## 8. MetaMask にローカルチェーンのネットワークを追加できない

- 症状: スマホ (または PC) の MetaMask に `http://...:8545` のネットワークを追加しようとすると、何度試しても拒否される
  - 原因: MetaMask は追加時に RPC へ実接続して検証します。接続先に届かないと拒否されます。とくにスマホからの場合、既定の `npx hardhat node` は `127.0.0.1` だけで待ち受けるため、スマホからは届きません。

### 確認・対処

- ノードを LAN 公開で起動し直します。

  ```bash
  npx hardhat node --hostname 0.0.0.0
  ```

- MetaMask に追加する RPC URL を **PC の LAN IP** にします (例: `http://192.168.1.20:8545`)。チェーン ID は `31337`。
- PC のファイアウォールで `8545` が許可されているか、PC とスマホが同じ Wi-Fi かを確認します。
- スマホ版フロントを使う場合は `iot-market-ui/.env.local` も同じ LAN IP に揃えます (項目7参照)。
