# Seller VC で出品身元を裏付ける (Stage 7)

!!! abstract "5 種目の VC: SellerVC"
    マーケット出品時に **「誰が」この dataset を売るのか**を VC で
    裏付ける層を追加します。Stage 1〜6 と同じパターンで提示 → トークン
    取得 → API 呼出。設計詳細は [SellerVC 設計仕様](../design/seller-vc-spec.md)。

## 目的

- ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC に続く **5 つ目の VC**
  (SellerVC) を体験する
- Seller が `IoTMarket.registerMerchandise()` で出品した直後、
  publisher に対して **`/marketplace/register`** を呼び、
  「この Merchandise は私が売っている」を VC で証明する
- Buyer が `/platform/data?merchandise=<addr>` を読むときに
  **`seller_did`** が返ってくる流れを確認する

## このページで分かること

- SellerVC の `licensed_datasets` で「この seller はどの dataset を売って良いか」
  を VC に焼き込む仕組み
- SellerToken (24h 多回利用) が `/marketplace/register` 専用で動くこと
- (任意) 環境変数 `MARKETPLACE_HARDHAT_RPC` 設定時、publisher が
  on-chain で `Merchandise.getOwner() == seller_eth_addr` を検証すること
- 監査ログに **`marketplace/seller_registered`** 行が追加されること
- buyer 側のレスポンスに **`seller_did` 同梱** で who-sold を可視化すること

## 全体像

```
[Seller wallet]
   │ 1. /issuer/offer?type=SellerVC → 受領
   │ 2. /verifier/request?vc_kind=SellerVC → 提示
   │ 3. SellerToken (24h 多回)
   │
[Seller (Hardhat console)]
   │ 4. Merchandise を deploy + IoTMarket.registerMerchandise()
   │
[Seller (curl or /seller UI)]
   │ 5. POST /marketplace/register
   │    Authorization: Bearer SellerToken
   │    body: merchandise_address + seller_eth_addr + tx_hash + dataset_id
   │
[publisher]
   │ - SellerToken 検証 (licensed_datasets に dataset_id が含まれる)
   │ - (任意) Merchandise.getOwner() を on-chain 検証
   │ - audit: raw_topic=marketplace/seller_registered
   │ - publisher 内部に merchandise → seller_did binding
   │
[Buyer (Stage 5/6 の購入動線)]
   │ → GET /platform/data?merchandise=<addr>
   │   レスポンスに seller_did が同梱される
```

## 前提

- Stage 5 / 6 のハンズオンを通せている
- iot-market-ui (Stage 6 ブランチ以降の deploy script で
  `dataset_id` を `additionalInfo` に持つ Merchandise が出る前提)
- publisher が Stage 7 / C2+C3 のコードで起動している
- iw3ip-wallet (iPhone) が動く
- LAN IP は `192.168.68.53` で示すので、あなたの環境で読み替え

---

## Step S0. 環境を最新コードに揃える

```bash
cd ~/program/Blockchain_IoT_Marketplace
git checkout main && git pull --ff-only

# (任意) on-chain owner verify を有効にする場合のみ
echo "MARKETPLACE_HARDHAT_RPC=http://host.docker.internal:8545" >> infra/.env

docker compose -f infra/docker-compose.yml --profile ssi-wallet --profile mv-bridge up --build -d
sleep 5

# 5 種目の VC が出ているか
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer \
  | python3 -m json.tool | grep -A1 SellerVC
# → "vct": "https://iw3ip.example/credentials/SellerVC/v1"
```

---

## Step S1. SellerVC を発行

### 何を確認するか
- `seller_id` と `licensed_datasets` (dataset_id のホワイトリスト) を query で渡し、
  wallet が SellerVC を受領する
- `licensed_datasets` に書いた dataset 以外は後で reject される

### 操作

PC ブラウザで:

```
http://192.168.68.53:8080/issuer/offer?type=SellerVC&seller_id=ertl-seller-001&licensed_datasets=home/env/temperature,home/env/humidity
```

QR を iPhone wallet で読み取り → **「IW3IP セラークレデンシャル」** 承認 → 受領。

### 期待結果
- wallet の VC 一覧に「IW3IP セラークレデンシャル」が追加
- claim:
    - `seller_id`: `ertl-seller-001`
    - `licensed_datasets`: `["home/env/temperature", "home/env/humidity"]`
    - `subject_id`: `did:jwk:...`

---

## Step S2. SellerVC を提示して SellerToken を取得

### 何を確認するか
- 提示で **SellerToken** が払い出される (PolicyToken/ViewerToken/ServiceToken
  でもなく、SellerToken)
- TTL は **24 時間 (86400 秒)**
- レスポンスに `licensed_datasets` がそのまま入る

### 操作

PC ブラウザで:

```
http://192.168.68.53:8080/verifier/request?vc_kind=SellerVC
```

`dataset_id` パラメータ不要 (SellerVC は dataset 非依存)。
QR → wallet で **SellerVC** を選択して提示。

### 期待出力

```bash
PUB=$(docker ps -qf name=publisher)
docker logs $PUB 2>&1 | grep "seller_token_issued" | tail -1
```

→
```
seller_token_issued jti=... token=... seller_id=ertl-seller-001 licensed=['home/env/temperature', 'home/env/humidity'] ttl=86400s
```

token を変数に:

```bash
SELLER=$(docker logs $PUB 2>&1 | grep "seller_token_issued" | tail -1 | sed -E 's/.*token=([^ ]+).*/\1/')
echo "SELLER=$SELLER"
```

---

## Step S3. Merchandise を deploy + IoTMarket に登録

### 何を確認するか
- Hardhat console で Merchandise を deploy し、`IoTMarket.registerMerchandise()` を呼ぶ
- deploy 時に `additionalInfo` に `dataset_id=home/env/temperature` を埋め込む
  (Stage 6 case B 互換)

### 操作

ターミナル B (Hardhat console):

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat console --network localhost
```

```javascript
// seller = Account #1 (iotOwner) を使う
const [_, iotOwner] = await ethers.getSigners();
console.log("seller:", await iotOwner.getAddress());

// PubKey 登録 (PubKey 初回利用時のみ)
const pubKey = await ethers.getContractAt("PubKey", "0x5FbDB2315678afecb367f032d93F642f64180aa3", iotOwner);
try { await (await pubKey.registerKey("[seller-key]")).wait(); } catch (e) { /* 登録済なら無視 */ }

// Merchandise を deploy (dataset_id を additionalInfo に含める)
const dataHash = "0x" + "5".repeat(64);
const merch = await (await ethers.getContractFactory("Merchandise", iotOwner)).deploy(
  ethers.parseEther("0.02"),
  dataHash,
  pubKey,
  [],   // deniedBuyers
  ["fileType", "dataSize", "dataset_id"],
  ["json", "1MB", "home/env/temperature"]
);
const deployReceipt = await merch.deploymentTransaction().wait();
const merchandiseAddress = await merch.getAddress();
console.log("merchandise:", merchandiseAddress);
console.log("deploy tx:", deployReceipt.hash);

// IoTMarket に登録
const market = await ethers.getContractAt("IoTMarket", "0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512", iotOwner);
const regTx = await market.registerMerchandise(merch);
const regReceipt = await regTx.wait();
console.log("register tx:", regReceipt.hash);
```

### メモ

- `seller`, `merchandise`, `deploy tx`, `register tx` の 4 つ

---

## Step S4. publisher に `/marketplace/register` で seller binding

### 何を確認するか
- Bearer SellerToken で `/marketplace/register` が成功する
- `licensed_datasets` に含まれる dataset_id だけ通る
- (任意) MARKETPLACE_HARDHAT_RPC 有効時、Merchandise.getOwner() が検証される
- audit log に `marketplace/seller_registered` 行が出る

### 操作

```bash
SELLER_ETH=<Step S3 のseller address>
MERCH=<Step S3 のmerchandise address>
TX_HASH=<Step S3 のregister tx hash>

curl -s -X POST http://192.168.68.53:8080/marketplace/register \
  -H "Authorization: Bearer $SELLER" \
  -H "Content-Type: application/json" \
  -d "{
    \"merchandise_address\": \"$MERCH\",
    \"seller_eth_addr\": \"$SELLER_ETH\",
    \"tx_hash\": \"$TX_HASH\",
    \"dataset_id\": \"home/env/temperature\"
  }" | python3 -m json.tool
```

### 期待出力

```json
{
  "registered": true,
  "merchandise_address": "0x...",
  "seller_did": "did:jwk:...",
  "dataset_id": "home/env/temperature",
  "tx_hash": "0x...",
  "register_count": 1,
  "owner_verify": "skipped"
}
```

`owner_verify` の値:
- `skipped`: `MARKETPLACE_HARDHAT_RPC` 未設定 (テスト/開発用)
- `verified`: on-chain で getOwner() が seller_eth_addr と一致
- `rpc_failed`: RPC に到達できず 502 (`fail-closed`)

### audit log

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=3' | python3 -m json.tool
```

→ 直近に:
```json
{
  "raw_topic": "marketplace/seller_registered",
  "subject_did": "did:jwk:...",
  "purpose": "register",
  "reason": "seller_register:jti=...:merchandise=0x...:eth=0x...:tx=0x...:owner_verify=skipped",
  "holder_did": "did:jwk:..."
}
```

---

## Step S5. Buyer から見える `seller_did`

### 何を確認するか
- Buyer が PurchaseViewerVC 経由で `/platform/data?merchandise=<addr>` を叩くと、
  レスポンスに **`seller_did`** が含まれる

### 操作

[Stage 5 ハンズオン](marketplace-vc-bridge.md) の手順で MetaMask 購入 → wallet 受領
→ ViewerToken 取得 → 取得。

```bash
TOKEN=<buyer の ViewerToken>
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://192.168.68.53:8080/platform/data?merchandise=$MERCH" | python3 -m json.tool
```

### 期待出力

```json
{
  "dataset_id": "home/env/temperature",
  "count": N,
  "read_count": 1,
  "seller_did": "did:jwk:...(seller の DID)",
  "rows": [...]
}
```

`seller_did` は Step S4 で binding した値です。Stage 5/6 だけ通した
Merchandise (`/marketplace/register` を呼んでいない) は `"unknown"` が返ります。

---

## Step S6. iot-market-ui の `/seller` ページから登録

### 何を確認するか
- ターミナルでなく Web UI からも `/marketplace/register` が叩けること
- フォームの 3 ステップ構造 (発行 → 提示 → 登録) が seller 視点で完結すること

### 操作

iot-market-ui を起動済の状態で:

```
http://192.168.68.53:5173/seller
```

UI の手順:

1. **Step 1**: `seller_id` と `licensed_datasets` を入れて
   「/issuer/offer を開く」 → wallet で受領
2. **Step 2**: 「/verifier/request を開く」 → wallet で SellerVC を提示 →
   publisher ログから `SellerToken` をコピー
3. **Step 3**: 上で取得した token + Step S3 の merchandise/eth/tx を貼り、
   `dataset_id` を選んで「Register」

成功時、画面下に Step S4 と同じ JSON レスポンスが表示されます。

---

## 完了判定マトリクス

| 工程 | 確認項目 | 出力例 |
| --- | --- | --- |
| S0 | 5 種類目の SellerVC が metadata に出る | `"vct": "...SellerVC/v1"` |
| S1 | wallet に SellerVC | claim に `seller_id`, `licensed_datasets` |
| S2 | SellerToken 発行 | `seller_token_issued ttl=86400s` |
| S3 | Merchandise + IoTMarket 登録 | tx hash 2 件 |
| S4 | `/marketplace/register` 成功 | `{registered: true, seller_did, register_count: 1}` |
| S4 | audit log | `raw_topic=marketplace/seller_registered`, `seller_register:...:owner_verify=...` |
| S5 | buyer side seller_did 表示 | `/platform/data` レスポンスに did:jwk: |
| S6 | UI 経由でも登録可 | `/seller` ページで成功 |

---

## トラブルシューティング

### A. `seller_token_dataset_not_licensed` で 403

**原因**: SellerVC の `licensed_datasets` に含まれない dataset を `/marketplace/register`
で渡した。

**対処**: Step S1 で `licensed_datasets` の中身を増やしておく、または対象 dataset の
SellerVC を別途発行する。

### B. `MARKETPLACE_HARDHAT_RPC` 有効化時に `chain_rpc_failed` (502)

**原因**: publisher コンテナから host.docker.internal:8545 に到達できない。

**対処**: Hardhat ノードが host で `--hostname 0.0.0.0` で起動しているか確認。
`infra/docker-compose.yml` の publisher service に `extra_hosts` 設定を加えた
状態で `docker compose up -d --build publisher` を再実行。

### C. `owner_mismatch` (403)

**原因**: SellerVC の holder と Merchandise.getOwner() の eth address が違う。
deploy した Hardhat account と SellerVC を取った eth account が違うときに起こる。

**対処**: 同じ Hardhat signer (例: Account #1 = `iotOwner`) で SellerVC 受領 +
Merchandise deploy をやり直す。

### D. その他

[Stage 5 のトラブルシューティング](marketplace-vc-bridge.md#トラブルシューティング)
が引き続き有効。

---

## 限界 / 将来課題

- **on-chain ガード**: 現状 `IoTMarket.registerMerchandise()` 自体は
  publisher 検証なしでも通る。on-chain でのガードは Stage 8+ 以降
- **EIP-712 署名**: SellerVC 経由の seller_did と eth address の紐付けは
  publisher が信頼する方式 (Stage 5/6 の eth_did_bound と同じ MVP レベル)
- **dataset_id をどこで宣言するか**: Stage 6 case B で `additionalInfo` の
  `dataset_id` 値を取れるようにしたが、`/marketplace/register` 側はクライアント
  POST に依存。将来は publisher 自身が `getAllAdditionalInfo()` を呼んで
  突合する強化案あり

## 関連

- [SellerVC 設計仕様](../design/seller-vc-spec.md)
- [Marketplace × Wallet bridge (Stage 5)](marketplace-vc-bridge.md)
- [Marketplace VC end-to-end (Stage 6)](marketplace-vc-end-to-end.md)
- [SSI Service (Stage 4 prep)](ha-ssi-service.md)
- [SSI Wallet (Stage 1)](ha-ssi-wallet.md)
