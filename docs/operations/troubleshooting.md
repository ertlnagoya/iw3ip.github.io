# トラブルシュート

よくある症状ごとに確認点と対応をまとめています。症状に近い項目を選び、上から順に確認してください。

## 1. `host.docker.internal` の接続エラー

- 症状: 仲介プロセス (`mediator-owner` / `mediator-buyer`) やフロントが `host.docker.internal` に接続できず、DNS 解決に失敗する
  - ログ例: `ConnectError("dns error", ... "failed to lookup address information: nodename nor servname provided, or not known")`
  - フロント例 (`iot-market-ui` の `VITE_RPC_URL` に `host.docker.internal` を設定している場合): `JsonRpcProvider failed to detect network ... (perhaps the URL is wrong or the node is not started)` が 1 秒おきに出続ける。既定の `http://127.0.0.1:8545` のままでこれが出るときは、この原因ではなくノードが止まっています (`npx hardhat node` のターミナルが動いているか確かめます)
  - 原因: `host.docker.internal` は Docker Desktop が **コンテナ内にのみ** 提供する特別なホスト名です。`cargo run` や `npm run dev` で **コンテナの外 (ホスト OS でネイティブ実行)** している場合、この名前は解決できません。

### 確認

- ノード自体が生きているかは、ホスト側で次を実行して確認します。

  ```bash
  curl http://localhost:8545
  ```

  `{"jsonrpc":"2.0", ... "error": ... "Parse error: Unexpected end of JSON input"}` が返れば、**ノードは到達可能で正常** です (空ボディの GET に対する JSON-RPC の正常な応答であり、エラーではありません)。

### 対処 A — hosts ファイルに追記する (推奨)

[最短起動の手順 0.5](../setup/quickstart.md) のとおり、hosts ファイルに `127.0.0.1 host.docker.internal` を追加します。`host.docker.internal` は設定ファイルのほか、`mediator-owner/src/main.rs` (IPFS と PostgreSQL の接続先)、`mediator-buyer/src/main.rs`、`iot-market-ui` の検索 API にも書かれているため、この方法がいちばん確実です。

### 対処 A' — 設定とソースを書き換える

hosts ファイルを変更できない場合は、上に挙げたファイル中の `host.docker.internal` を `localhost` に書き換えます。設定ファイルは次のとおりです。

- `mediator-owner/settings/owner_1.yaml`:

  ```yaml
  api_url: "http://localhost:3000"
  rpc_url: "http://localhost:8545"
  ```

- 購入者側 `mediator-buyer` (設定ファイルは無く、`mediator-buyer/src/main.rs` の定数 `API_URL` と `RPC_URL` に書かれている)、および、フロント (`iot-market-ui`) の `src/routes/api/sql-query/+server.ts` (IPFS と PostgreSQL の接続先が `host.docker.internal` に固定されている) にも同様に適用します。フロントの RPC 接続先は、既定で `http://127.0.0.1:8545` なので変更不要です。接続先が見当たらない場合は、リポジトリ内を `host.docker.internal` で grep して洗い出してください。
- 変更後、各プロセスを再起動します。

### 対処 B — コンテナ内で動かす

仲介プロセスやフロントを DevContainer / Docker 側で実行すれば、`host.docker.internal` がそのまま解決します。設定を変更したくない場合はこちらを選びます。

## 2. IPFS コンテナが再起動を繰り返す

- 症状: `ipfs` ディレクトリで `docker compose ps` を実行すると、`ipfs` の STATUS が `Restarting` のままになる
  - 確認: `docker compose logs ipfs` でエラーの内容を見ます
  - 対応: 保存データが壊れている場合は、`docker compose down -v` でボリュームごと削除し、`docker compose up -d` で作り直します。IPFS と PostgreSQL のデータは消えます。`ipfs/README.md` の手順で `ipfs_records` テーブルを作り直し、`mediator-owner` を起動し直して登録し直します
- 症状: `docker compose up -d` で `Bind for 0.0.0.0:8080 failed: port is already allocated` と出て、IPFS が起動しない
  - 確認: publisher が動いていないか確認します。どちらもポート 8080 を使います ([10. ポートが競合する](#10-ポートが競合する))
  - 対応: publisher を止めてから (`cd ~/program/Blockchain_IoT_Marketplace && docker compose -f infra/docker-compose.yml stop publisher`)、`ipfs` ディレクトリで `docker compose up -d` をもう一度実行します。データを消す必要はありません

## 3. スマホから `http://<HOST_IP>:5173/` が開けない

- 症状: スマホのブラウザで `http://<HOST_IP>:5173/` (`<HOST_IP>` は PC の LAN IP) を開けない
  - 確認: `npm run dev -- --host 0.0.0.0 --port 5173` で起動したか
  - 確認: PC とスマホが同一ネットワークか
  - 確認: PC のファイアウォールで `5173` が遮断されていないか

## 4. 商品が表示されない

- 症状: <http://localhost:5173> を開いても商品が表示されない
  - トップページは検索画面です。「Run Search」で一覧に出るのは、`mediator-owner` が動画のメタデータを IPFS と PostgreSQL (`ipfs_records` テーブル) に登録した商品だけです。イベントファイルから作られた商品や、デプロイで登録した 5 件は検索に出ません。`http://localhost:5173/merchandise/<商品のアドレス>` を直接開いて確認します。
  - 確認: `mediator-owner` の実行引数が `cargo run -- settings/owner_1.yaml` になっているか
  - 確認: `mediator-owner` のターミナルに `Watching raw_data/output for new event files` と表示されているか。表示されない場合は、教材リポジトリを最新にして (`git pull`) 起動し直します
  - 確認: イベントファイルを出力したとき、`Product deployed: 0x... for file ...` と表示されるか。表示されたアドレスが商品ページの URL になります

## 5. 購入できない

- 症状: 購入できない
  - 確認: MetaMask の接続アカウントが Account #2 (`0x3C44...`) か。購入には、購入者の公開鍵が PubKey コントラクトに登録されている必要があります。公開鍵が登録されているのは、`mediator-buyer` が起動時に登録する Account #2 と、`mediator-owner` が登録する Account #1 だけです。ほかのアカウントでは購入が取り消されます (revert)。Account #1 では購入は通りますが、データは届きません
  - 確認: その商品を、同じアカウントですでに購入していないか。同じアカウントは同じ商品を一度しか買えません (もう一度買うと取り消されます)。また、受け渡しの途中 (購入してから、データの受け渡しが終わるまで) の商品は、ほかのアカウントでも買えません。`mediator-owner` のターミナルに出た、まだ買っていない `Product deployed: 0x...` の商品を使うか、ノードを起動し直してデプロイし直します
  - 対応: ローカルチェーン再起動後は MetaMask のキャッシュをクリア

## 6. イベントが共有されず `no_matching_consent` で拒否される

- 症状: HA Demo Simulator などで、一部のデータセットだけ `/platform/ingest` に出てこない。`audit/logs` を見ると `action: deny` / `reason: no_matching_consent` / `subject_did: "unknown"` になっている
  - ログ例:

    ```json
    {"action":"deny","subject_did":"unknown","dataset_id":"home/event/possible_littering","purpose":"research","reason":"no_matching_consent", ...}
    ```

  - 原因の候補: consent を登録していない、または **consent の有効期限切れ**。consent ファイル (`examples/ha_demo/consent_*.json`) の `valid_to` が過去の日付だと、登録しても現在時刻が有効期間外になり、一致せず拒否されます。教材のサンプルは `2030-12-31T23:59:59Z` まで有効ですが、日付を書き換えたファイルや自分で作ったファイルでは期限切れが起こりえます。

### 確認

- まず audit ログを読みます (zsh では `?` がワイルドカード扱いになるため URL をクォートで囲みます)。

  ```bash
  curl 'http://localhost:8080/audit/logs?limit=20'
  ```

- 拒否されている `dataset_id` の consent ファイルを開き、`valid_from` 〜 `valid_to` の範囲に**今日の日付**が入っているか確認します。`valid_to` が過去なら期限切れです。

### 対処

- consent ファイルの `valid_to` を先の日付に延ばし、登録し直します。すべてのファイルの `valid_to` をまとめて書き換える例 (macOS の `sed` は `-i ''` が必要):

  ```bash
  cd ~/program/Blockchain_IoT_Marketplace
  sed -i '' 's/"valid_to": "[^"]*"/"valid_to": "2035-12-31T23:59:59Z"/' examples/ha_demo/consent_*.json
  ```

- 編集後、consent を登録し直し (各レスポンスが `"status":"stored"` になることを確認)、イベントを送り直してから `/platform/ingest` を再確認します。
- 既存の `deny` ログは履歴として残ります。必ず **consent 登録後に送り直した新しいイベント** で結果を確認してください。

## 7. 商品ページが 500 になる (iot-market-ui の接続先)

- 症状: 一覧 (`/`) は開けるのに、個別の商品ページ `http://localhost:5173/merchandise/<address>` を開くと **HTTP 500** になる
  - 原因: フロント (`iot-market-ui`) は RPC と publisher の接続先を `iot-market-ui/.env.local` から読み込みます。この接続先が **起動中の Hardhat ノードに届かない値** (他人の IP、`8546` などの誤ポート、未起動のホスト) だと、商品ページのデータ取得が失敗し 500 になります。一覧ページはタイムアウト処理があるため描画はされますが、商品詳細ページは失敗がそのまま 500 として表面化します。

### 確認・対処

- `iot-market-ui/.env.local` を開き、`VITE_RPC_URL` が起動中のノード (既定は `http://127.0.0.1:8545`) を指しているか確認します。ファイルが無ければ雛形からコピーします。

  ```bash
  cd ~/program/Blockchain_IoT_Marketplace/iot-market-ui
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
  # 動いているノード (最短起動のターミナル 1) を Ctrl+C で止めてから
  cd ~/program/Blockchain_IoT_Marketplace/iot-market
  npx hardhat node --hostname 0.0.0.0
  ```

  起動し直すとチェーンが空になります。[最短起動](../setup/quickstart.md) の手順 2 (デプロイ) をやり直し、`mediator-owner` と `mediator-buyer` も再起動してください。MetaMask で nonce のエラーが出たら、アカウントのリセット (Settings → Advanced → Clear activity tab data) を行います。

- MetaMask に追加する RPC URL を **PC の LAN IP** にします (例: `http://192.168.1.20:8545`)。チェーン ID は `31337`。
- PC のファイアウォールで `8545` が許可されているか、PC とスマホが同じ Wi-Fi かを確認します。
- スマホ版フロントを使う場合は `iot-market-ui/.env.local` も同じ LAN IP に揃えます (項目7参照)。

## 9. `GET /platform/ingest` が 404 になる

- 症状: `curl http://localhost:8080/platform/ingest` が `{"detail":"platform_ingest_read_disabled"}` (404) を返す
  - 原因: `GET /platform/ingest` は認証なしで全件を返すデモ用の API で、既定では無効です。
  - 対応: `PLATFORM_INGEST_READ_ENABLED=true` を付けて publisher を起動し直します。

    ```bash
    cd ~/program/Blockchain_IoT_Marketplace
    PLATFORM_INGEST_READ_ENABLED=true docker compose -f infra/docker-compose.yml up --build -d
    ```

## 10. ポートが競合する

- 症状: `address already in use` や `port is already allocated` と出て起動しない
  - 確認: 使用中のプロセスを調べます (macOS / Linux)。

    ```bash
    lsof -i :8080
    ```

  - 教材で使う主なポートは次のとおりです。

    | ポート | 使うもの | 注意 |
    |---|---|---|
    | 8080 | IPFS のゲートウェイ (`ipfs/docker-compose.yaml`) と publisher (`infra/docker-compose.yml`) | 同時には使えない。publisher を使う前に `ipfs` ディレクトリで `docker compose down` を実行する |
    | 8545 | Hardhat ノード (`npx hardhat node`) | 2 つ目のノードを起動しない |
    | 5173 | フロントエンド (`iot-market-ui`、`npm run dev`) | 別の Vite 開発サーバが動いていないか確認する |
    | 8090 | assistant (`assistant` / `assistant-llm` / `assistant-demo` のプロファイル) | 3 つのプロファイルは同じポートを使うので、1 つずつ起動する |
    | 18000 | llm-mock (`llm-mock` / `assistant-demo` のプロファイル) | ホストで `uvicorn examples.phase3_llm_mock_server:app --port 18000` を動かす前に、`llm-mock` / `assistant-demo` のコンテナを止める |
    | 11434 | Ollama (PC に直接入れた場合) | Ollama アプリが常駐していると、別の Ollama を起動できない |

  - 対応: 競合しているプロセスを止めてから起動し直します。
