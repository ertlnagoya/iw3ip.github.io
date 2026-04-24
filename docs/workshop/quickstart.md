# 最短起動

この章は「まず全体を動かす」ための最短手順です。

## 方針

- 深い理解より「まず全体を起動して観察する」ことを優先
- 途中で止まったら [トラブルシュート](../operations/troubleshooting.md) を参照
- ブロックチェーンの基礎が不安なら先に [Hardhat基礎](../foundations/hardhat-basics.md) を読む
- 実機や複数プロセスの起動が難しい場合は [Home Assistant Demo Simulator サンプル](../hands-on/ha-demo-simulator.md) で代替可

## 起動順

1. `iot-market`（ローカルチェーン + デプロイ）
2. `iot-market-ui`（フロントエンド）
3. `simple-storage`（データ保存）
4. `ipfs`（IPFS + PostgreSQL）
5. `mediator-owner`
6. `mediator-buyer`

## 主要コマンド

### 1. ローカルブロックチェーンを起動

`iot-market` は Hardhat を使ってローカルチェーンを立ち上げます。  
参考: <https://hardhat.org/docs>

```bash
cd iot-market
npx hardhat node
```

別ターミナル:

### 2. コントラクトをデプロイ

```bash
cd iot-market
npx hardhat run scripts/deployMerchandiseWithIoTMarket.ts --network localhost
```

### 3. フロントエンドを起動

```bash
cd iot-market-ui
npm run dev
```

### 4. ストレージサーバーを起動

```bash
cd simple-storage
cargo run
```

### 5. IPFS と PostgreSQL を起動

```bash
cd ipfs
docker compose up -d
```

### 6. オーナー側プロセスを起動

```bash
cd mediator-owner
cargo run -- settings/owner_1.yaml
```

### 7. 購入者側プロセスを起動

```bash
cd mediator-buyer
cargo run --bin mediator-b
```

## 成功判定

- フロントが開く: `http://localhost:5173`
- 商品一覧が取得できる
- エラーで停止しているプロセスがない

## よくあるつまずき

- `npx: command not found`
  - Node.js が正しく入っていません。<https://nodejs.org/> を確認してください。
- `cargo: command not found`
  - Rust が未インストールです。<https://www.rust-lang.org/tools/install> を参照してください。
- MetaMask から購入できない
  - ローカルネットワーク設定とアカウントを確認してください。
  - 参考: [MetaMask](https://metamask.io/)

## メタマスク設定イメージ

![MetaMask Network 設定例](../assets/how_to_network.png)
