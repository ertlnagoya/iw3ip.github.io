# トラブルシュート

よくある症状ごとに確認点と対応をまとめています。症状に近い項目を選び、上から順に確認してください。

## 1. `host.docker.internal` の接続エラー

- 症状: `host.docker.internal` に接続できない
  - 確認: DevContainer / Docker Desktop 前提で起動しているか
  - 確認: 直接ホスト実行している場合はネットワーク設定が適切か

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
