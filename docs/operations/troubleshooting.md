# トラブルシュート

## 1. `host.docker.internal` の接続エラー

- 症状: `host.docker.internal` に接続できない
  - 確認: DevContainer / Docker Desktop 前提で起動しているか
  - 確認: 直接ホスト実行している場合はネットワーク設定が適切か

## 2. IPFSコンテナが再起動ループ

- 症状: IPFS コンテナが再起動を繰り返す
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

## 5. 購入できない

- 症状: 購入できない
  - 確認: MetaMask の接続アカウントが正しいか
  - 対応: ローカルチェーン再起動後は MetaMask のキャッシュをクリア
