# Hardhat基礎（このサイトで使うブロックチェーン）

## 一般的な説明

### Hardhatとは

Hardhatは、Ethereum系スマートコントラクトの開発・テスト・デプロイを行うための開発環境です。

このサイトでは、**ローカル開発チェーン**として使い、実験を安全かつ再現可能に行います。

### 何ができるか

- ローカルノード起動（`npx hardhat node`）
- コントラクトデプロイ（`npx hardhat run ... --network localhost`）
- テスト実行（`npx hardhat test`）

### なぜ学習に向いているか

- 手元PCだけでブロックチェーン実験ができる
- テスト用アカウントと残高が最初から用意される
- デプロイ・呼び出し・イベント確認を短いサイクルで試せる

### 開発フロー（最小）

```mermaid
sequenceDiagram
  participant Dev as Developer
  participant HH as Hardhat Node
  participant SC as Smart Contract
  Dev->>HH: start local node
  Dev->>SC: deploy script
  Dev->>SC: call contract functions
  SC-->>Dev: tx receipts / events
```

## 本システムでの位置付け

### IW3IPにおける役割

- 学習段階: Hardhatで「契約実行の流れ」を理解
- 実運用検討: 公開チェーン/許可型チェーンの選定と運用設計へ展開

### このサイトで実際に触るもの

- `npx hardhat node`: ローカルチェーンの起動
- デプロイスクリプト: コントラクトをローカルネットワークへ配置
- MetaMask: ローカルチェーンへ接続し、UIから取引を送る

### よくあるつまずき

- MetaMaskのネットワーク状態が古い -> アカウント/ネットワークを再同期
- デプロイスクリプト失敗 -> ノード再起動後に再デプロイ
- ポート衝突 -> `8545` 利用状況を確認

### 実運用との違い

- Hardhat はあくまで開発・学習用
- 本番では、ネットワーク運用、ガスコスト、障害対応、秘密鍵管理を別途考える必要がある

## 出典

- Hardhat official docs: <https://hardhat.org/docs>
- Hardhat getting started: <https://hardhat.org/hardhat-runner/docs/getting-started>
