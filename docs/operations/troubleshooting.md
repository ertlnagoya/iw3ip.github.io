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
