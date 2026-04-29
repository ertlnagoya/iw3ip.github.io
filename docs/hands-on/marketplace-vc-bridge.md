# マーケットプレイス × ウォレット連携 (v2 / Stage 5)

!!! abstract "v2 を体験するハンズオン"
    マーケットプレイスでの購入を起点に、買い手のウォレットへ
    PurchaseViewerVC を発行し、提示によってデータを取得する
    end-to-end フローを体験します。
    設計詳細は [Marketplace VC Bridge 設計仕様](../design/marketplace-vc-bridge-spec.md)
    を参照。

!!! tip "dataset の選択"
    例は `home/env/temperature` で書かれていますが、Stage 6 case B 以降の
    deploy script では Merchandise #2 (`home/event/possible_littering`) と
    Merchandise #4 (`home/event/flood_risk_high`) も登録されており、
    Stage 0 と同じイベントで購入動線を試せます
    ([purchase-viewer-possible-littering.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/ssi_wallet/purchase-viewer-possible-littering.json)
    の PD 登録済)。

## 目的

このハンズオンでは、**v1 (現行マーケット + MetaMask + 暗号化 IPFS 配信)**
の上に **v2 (VC 連携)** が乗ったときに、認可がどう変わるかを体験します。

- 同じ人間が **2 つの身元** (ETH 鍵と did:jwk) を扱う意味
- ETH 支払いと VC 認可の **役割分離** (支払いと閲覧権限が別レイヤー)
- bridge service が on-chain Purchase event を off-chain VC 発行に
  橋渡しする仕組み

## このページで分かること

- v1 (`emitUpload(encryptURI)` 復号) と v2 (`PurchaseViewerVC` 経由)
  が **同一購入から並走**する動き
- `/marketplace/claim` が冪等 (同じ tx_hash を投げても 1 件しか発行されない)
- audit log に `eth_addr ↔ did:jwk` 紐付けが記録される
- `/platform/data?merchandise=<addr>` が ViewerToken で gate される

## 全体像

```
[buyer]
   │  iot-market-ui で商品を選び、MetaMask で purchase()
   ▼
[Merchandise.purchase tx]
   │  ↓                        ↓
   │  Purchase event          (任意) v1 lane: encryptURI 配信
   ▼
[bridge listener]   または    [iot-market-ui /purchased/[txHash]]
   │                           │
   └───────┬───────────────────┘
           │  POST /marketplace/claim (冪等 on tx_hash)
           ▼
       [publisher]
           │  pre_authorized_code を発行、deeplink を返す
           │  audit: marketplace/claim
           ▼
       [iw3ip-wallet]
           │  deeplink で OID4VCI 受領
           │  → PurchaseViewerVC (claims: merchandise/tx/eth)
           │  audit: marketplace/issued (eth_did_bound)
           ▼
       [iw3ip-wallet]
           │  OID4VP で提示
           │  → ViewerToken (60s 多回利用)
           ▼
       [/platform/data?merchandise=<addr>]
           │  ViewerToken Bearer で取得
           │  audit: platform/data (viewer_token_used)
           ▼
       [JSON データ取得]
```

## 前提

- [Stage 1](ha-ssi-wallet.md) と [Stage 3](ha-ssi-viewer.md) の動作確認が済んでいる
- iot-market-ui (Svelte) と Hardhat ローカルチェーンが起動できる
- iw3ip-wallet が iPhone 実機で動く (Metro bundler 接続済み)
- LAN IP を確認 (`ipconfig getifaddr en0`)。本ページでは `192.168.68.53` で示すので、
  あなたの環境の IP に読み替えてください

---

## Step 1. Hardhat ノード起動

### 何を確認するか
- Hardhat ローカルチェーンが起動し、20 個のテストアカウントが利用可能になる
- buyer 用に Account #2 の秘密鍵をメモする (後で MetaMask に取り込む)

### 操作

ターミナル A (起動したまま):

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat node --hostname 0.0.0.0
```

### 期待出力 (一部抜粋)

```
Started HTTP and WebSocket JSON-RPC server at http://0.0.0.0:8545/

Accounts
========
Account #0: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (10000 ETH)
Private Key: 0xac0974bec...

Account #2: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC (10000 ETH)
Private Key: 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a
...
```

**メモ**: Account #2 の秘密鍵 (`0x5de4...`)、Account #2 のアドレス (`0x3C44...`)。

---

## Step 2. コントラクトデプロイ

### 何を確認するか
- PubKey + IoTMarket + Merchandise×5 が一括デプロイされる
- IoTMarket と最初の Merchandise のアドレスをメモする (毎回固定値が出る)

### 操作

ターミナル B:

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat run scripts/deployMerchandiseWithIoTMarket.ts --network localhost
```

### 期待出力

```
Contract "PubKey" with 0x5FbDB2315678afecb367f032d93F642f64180aa3 deployed
Contract "IoTMarket" with 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 deployed
Contract "Merchandise" with 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0 deployed
Merchandise 0 registered
Contract "Merchandise" with 0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9 deployed
Merchandise 1 registered
Contract "Merchandise" with 0x0165878A594ca255338adfa4d48449f69242Eb8F deployed
Merchandise 2 registered
Contract "Merchandise" with 0x2279B7A0a67DB372996a5FaB50D91eAA73d2eBe6 deployed
Merchandise 3 registered
Contract "Merchandise" with 0x610178dA211FEF7D417bC0e6FeD39F05609AD788 deployed
Merchandise 4 registered
```

**メモ**: IoTMarket = `0xe7f1725...` (これはハンズオン中で **何度も使う**)。
Merchandise アドレス 5 つ (購入時に 1 つずつ消費する)。

---

## Step 3. PubKey 登録 (buyer の事前準備)

### 何を確認するか
- `Merchandise.purchase()` 内部で `i_pubKey.getPubKey(tx.origin)` が呼ばれるため、
  購入する account (Account #2) は事前に **PubKey** コントラクトへの公開鍵登録が必要
- 登録漏れだと購入時に `PubKey__NotRegistered` で revert する

### 操作

ターミナル B (Hardhat console):

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat console --network localhost
```

console プロンプトで:

```javascript
const [marketOwner, iotOwner, buyer] = await ethers.getSigners();
const pubKey = await ethers.getContractAt(
  "PubKey",
  "0x5FbDB2315678afecb367f032d93F642f64180aa3",
  buyer
);
const tx = await pubKey.registerKey("[dummy-pubkey-for-handson]");
await tx.wait();
console.log("registered for:", await buyer.getAddress());
.exit
```

### 期待出力

```
registered for: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
```

**形式**: `[` で始まり `]` で終わる文字列なら何でも可 (`isPubKey()` が形式チェックのみ)。
本来は RSA 公開鍵 (v1 lane で seller が暗号化に使う) ですが、ハンズオンでは v2 lane が
本筋なので dummy で十分です。

---

## Step 4. publisher + bridge を起動

### 何を確認するか
- publisher が PurchaseViewerVC を含む VC 種を発行できる状態 (Stage 7 まで通っていれば 5 種すべてが揃う)
- bridge が Hardhat の Purchase event を購読し、Merchandise×5 を見ている

### 操作

ターミナル C:

```bash
cd ~/program/Blockchain_IoT_Marketplace
git checkout main && git pull --ff-only

# bridge 用 env (LAN IP は ipconfig getifaddr en0 で確認した値)
cat > infra/.env <<EOF
BRIDGE_HARDHAT_RPC=http://host.docker.internal:8545
BRIDGE_IOT_MARKET_ADDRESS=0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
BRIDGE_PUBLIC_PUBLISHER_URL=http://192.168.68.53:8080
EOF

docker compose -f infra/docker-compose.yml \
  --profile ssi-wallet --profile mv-bridge \
  up --build -d
```

### 期待出力 (確認 3 件)

#### 4-A. publisher ヘルスチェック

```bash
sleep 5
curl -s http://192.168.68.53:8080/health
```
→ `{"status":"ok","service":"publisher"}`

#### 4-B. PurchaseViewerVC が登録済か

```bash
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer | python3 -m json.tool | grep -A1 PurchaseViewerVC
```
→
```
"PurchaseViewerVC": {
  "format": "dc+sd-jwt",
  ...
"vct": "https://iw3ip.example/credentials/PurchaseViewerVC/v1",
```

#### 4-C. bridge が Hardhat に接続して Merchandise を発見しているか

```bash
docker logs iw3ip-mv-bridge 2>&1 | tail -5
```
→
```
bridge: listening to 5 merchandise(s)
bridge: starting poll from block 14
bridge: started rpc=http://host.docker.internal:8545 market=0xe7f1725... publisher=http://publisher:8080
```

!!! warning "BRIDGE_PUBLIC_PUBLISHER_URL の設定は必須"
    これがないと、後の Step 6 で発行される deeplink 内の
    `credential_issuer` が Docker 内部ホスト名 `http://publisher:8080` になり、
    iPhone wallet が deeplink を開いた瞬間にメタデータ fetch で詰まります。

---

## Step 5. MetaMask 設定

### 何を確認するか
- MetaMask が Hardhat localhost (chain ID 31337) に接続できる
- buyer (Account #2) の秘密鍵がインポートされ 10000 ETH が見える

### 操作

ブラウザは **Chrome / Firefox / Brave 等の MetaMask 拡張対応**を使用 (Safari 不可)。

1. MetaMask 拡張アイコン → 上部のネットワーク名 → **「ネットワークを追加」**
2. 入力:
   - Network Name: `Hardhat localhost`
   - RPC URL: `http://192.168.68.53:8545`
   - Chain ID: `31337`
   - Currency: `ETH`
3. 保存後、`Hardhat localhost` に切替
4. アカウントメニュー → **「秘密鍵をインポート」** → Step 1 でメモした Account #2 の秘密鍵

### 期待表示

- ネットワーク: `Hardhat localhost`
- アドレス: `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`
- 残高: `10000 ETH`

---

## Step 6. 購入 → bridge が Purchase event を捕捉

### 何を確認するか
- MetaMask 経由で `Merchandise.purchase()` が成功する
- bridge が **Purchase event を polling で検知**して `/marketplace/claim` を叩く
- publisher の audit log に `marketplace/claim` 行が新規で残る

### 操作 A: ブラウザ経由 (本筋)

iot-market-ui を起動 (ターミナル D):

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market-ui
cat > .env.local <<EOF
VITE_RPC_URL=http://192.168.68.53:8545
VITE_PUBLISHER_URL=http://192.168.68.53:8080
EOF
npm run dev -- --host 0.0.0.0 --port 5173
```

PC ブラウザ (Chrome 等) で次にアクセス:
```
http://192.168.68.53:5173/merchandise/0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9
```

(Merchandise #1。`#0` は使わなくても良いが state が IN_PROGRESS のまま残るので注意)

「Purchase」→ MetaMask で確認 → tx 送信。確定後、自動で
`/purchased/<txHash>?merchandise=...&dataset=...&buyer=...` に遷移し、QR と deeplink が出る。

### 操作 B: MetaMask が動かない場合の fallback

ターミナル B (Hardhat console):

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat console --network localhost
```

```javascript
const [marketOwner, iotOwner, buyer] = await ethers.getSigners();
const merch = await ethers.getContractAt(
  "Merchandise",
  "0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9",
  buyer
);
const tx = await merch.purchase({ value: await merch.getPrice() });
const r = await tx.wait();
console.log("tx hash:", r.hash, "status:", r.status);
```

→ `tx hash: 0x4e7c...` `status: 1`

### 期待出力 (5 秒待ってから確認)

#### 6-A. bridge ログで Purchase event を捕捉

```bash
docker logs iw3ip-mv-bridge 2>&1 | tail -5
```
→
```
bridge: Purchase event from 0xDc64a140... buyer=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC tx=0x4e7c3e0a...
bridge: claim ok jti=2b417b32e6566830 deeplink=openid-credential-offer://?credential_offer=...
```

ここの `jti=...` の値はあとで使うのでメモ。

#### 6-B. audit log に marketplace/claim 行

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=2' | python3 -m json.tool
```
→
```json
{
  "raw_topic": "marketplace/claim",
  "subject_did": "eth:0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "purpose": "purchase",
  "reason": "claim_received:2b417b32e6566830:tx=0x4e7c3e0a...",
  ...
}
```

**ここまでで「on-chain 購入 → off-chain 認可コンテキスト確保」が成立**。

---

## Step 7. iPhone wallet で PurchaseViewerVC を受領

### 何を確認するか
- bridge の deeplink を iPhone wallet (iw3ip-wallet) で開いて VC 発行を受ける
- VC の claims に `merchandise_address` / `tx_hash` / `buyer_eth_addr` が乗っている
- publisher が **eth_addr ↔ did:jwk の紐付けを audit log に記録する**

### 事前準備: Metro bundler 起動

iPhone の wallet が `No script URL provided` エラーで開けない場合、Metro が起動していない。
ターミナル E:

```bash
cd ~/program/iw3ip-wallet
npx react-native start
```

`Metro waiting on...` を確認。wallet で「Reload JS」を押せば普通の画面に戻る。

### 操作: deeplink を QR で iPhone に渡す

```bash
DEEPLINK=$(docker logs iw3ip-mv-bridge 2>&1 | grep "claim ok jti=2b417b32e6566830" | tail -1 | sed -E 's/.*deeplink=//')
echo "$DEEPLINK"

# QR を Mac ブラウザで開く
ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$DEEPLINK")
open "https://api.qrserver.com/v1/create-qr-code/?size=400x400&data=$ENCODED"
```

iPhone カメラで QR を読み取り → wallet が起動 → 「IW3IP Purchase Viewer Credential」承認画面 → 承認。

### 期待結果

#### 7-A. wallet の VC 一覧で claims を展開すると次が見える

- `dataset_id`: `home/env/temperature`
- `merchandise_address`: `0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9`
- `buyer_eth_addr`: `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`
- `tx_hash`: `0x4e7c3e0a...`
- `allowed_actions`: `["read"]`

#### 7-B. audit log で eth↔did 紐付けが記録される

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=3' | python3 -m json.tool
```
→
```json
{
  "raw_topic": "marketplace/issued",
  "subject_did": "did:jwk:eyJhbG...",
  "purpose": "purchase_link",
  "reason": "eth_did_bound:claim=2b417b32...:eth=0x3C44...:tx=0x4e7c3e0a...",
  "holder_did": "did:jwk:eyJhbG..."
}
```

**この `eth_did_bound` 行が Stage 5 のキーマイルストーン**。
MetaMask の鍵と wallet の鍵が「同じ人物」として publisher 上で結びつきました。

---

## Step 8. ViewerToken を取り出す

### 何を確認するか
- wallet で **PurchaseViewerVC を提示**すると ViewerToken (60 秒・多回利用) が払い出される
- PolicyToken (Stage 1) でも ViewerVC (Stage 3) でもなく、購入連動の VC を選ぶ動線

### 操作

PC ブラウザで:
```
http://192.168.68.53:8080/verifier/request?dataset_id=home/env/temperature&vc_kind=PurchaseViewerVC
```

`vc_kind=PurchaseViewerVC` が必須 (これが無いと ConsentVC 用 PD が選ばれる)。

QR / deeplink → iPhone wallet で **PurchaseViewerVC を選択して提示**。

publisher ログから token 抽出:

```bash
PUB=$(docker ps -qf name=publisher)
TOKEN=$(docker logs $PUB 2>&1 \
  | grep "viewer_token_issued vc_kind=PurchaseViewerVC" | tail -1 \
  | sed -E 's/.*token=([^ ]+).*/\1/')
echo "TOKEN=$TOKEN"
```

### 期待出力

```
TOKEN=oJVsNtb5Un1NginSyyCcJavThu9WkTxRT6b8uhmWjRc
```

(JSON 形式のログ全文も出ます: `viewer_token_issued vc_kind=PurchaseViewerVC jti=cc38f06e... token=... dataset=home/env/temperature ttl=60s`)

---

## Step 9. データ取得 (`merchandise=<addr>` 逆引き)

### 何を確認するか
- ViewerToken で `/platform/data` が叩ける
- `merchandise=<contract address>` を query に渡すと、publisher が **dataset_id を逆引き**して同じ結果を返す
- 期限切れ (60 秒経過) で 401 になる

### 操作 (60 秒以内)

```bash
MERCHANDISE=0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9

curl -s -H "Authorization: Bearer $TOKEN" \
  "http://192.168.68.53:8080/platform/data?merchandise=$MERCHANDISE" | python3 -m json.tool
```

### 期待出力

```json
{
  "dataset_id": "home/env/temperature",
  "count": 0,
  "read_count": 1,
  "rows": []
}
```

`count: 0` でも認可は通っているので OK。実データ投入は別途 `POST /platform/ingest`
(Stage 1) で行えますが、本ハンズオンでは認可ロジックの検証が目的。

### 期限切れの確認 (60 秒経過後)

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://192.168.68.53:8080/platform/data?dataset_id=home/env/temperature" | python3 -m json.tool
```
→
```json
{
  "detail": "viewer_token_expired"
}
```

これも Stage 5 / Stage 3 仕様通りの正常動作です。

---

## Step 10. 監査ログの全体像

### 何を確認するか
- 1 購入から **少なくとも 4 行**の audit log が連鎖して残る
- on-chain (ETH 鍵) と off-chain (did:jwk) の対応関係が読み取れる

### 操作

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=10' | python3 -m json.tool
```

### 期待出力 (新しい順)

| id | raw_topic | reason | subject |
| --- | --- | --- | --- |
| (最新) | `platform/data` | `viewer_token_used:cc38f06e...:1` | `did:jwk:...` |
| -1 | `oid4vp/response` | `ok` | `did:jwk:...` |
| -2 | `marketplace/issued` | **`eth_did_bound:claim=2b417b32...:eth=0x3C44...:tx=0x4e7c...`** | `did:jwk:...` |
| -3 | `marketplace/claim` | `claim_received:2b417b32...:tx=0x4e7c...` | `eth:0x3C44...` |

`marketplace/claim` (eth_addr 主体) と `marketplace/issued` (did:jwk 主体) が **同じ claim_id でリンク**しているのが Stage 5 の核心です。

---

## v1 lane との対比

同じ購入で v1 lane (encryptURI 配信) も並走しています。
seller 側で `Merchandise.emitUpload(encryptURI)` を呼んでいれば、
buyer の MetaMask 側でも従来通りの暗号化 URI が見えます。

| 観点 | v1 (`emitUpload`) | v2 (`PurchaseViewerVC`) |
| --- | --- | --- |
| 配信形態 | encryptURI (公開鍵暗号) | publisher API (Bearer) |
| 鍵の管理 | MetaMask の ETH 鍵 | wallet の did:jwk |
| データ取得元 | IPFS (or 任意) | publisher の `/platform/data` |
| 監査ログ | on-chain Upload event のみ | publisher audit log (eth_did_bound 込み) |
| 撤回 | 鍵の漏洩で再暗号化 | wallet で VC 破棄 |

---

## 完了判定マトリクス

下表が全部 ✅ になれば Stage 5 完了です。

| 工程 | 確認項目 | 出力例 / 場所 |
| --- | --- | --- |
| Step 1 | Hardhat ノード起動 | `Started HTTP and WebSocket JSON-RPC server at http://0.0.0.0:8545/` |
| Step 2 | コントラクトデプロイ | `Contract "IoTMarket" with 0xe7f1725...` |
| Step 3 | PubKey 登録 | `registered for: 0x3C44...` |
| Step 4-A | publisher 起動 | `{"status":"ok","service":"publisher"}` |
| Step 4-B | PurchaseViewerVC が露出 | issuer metadata に `"vct": ".../PurchaseViewerVC/v1"` |
| Step 4-C | bridge が Hardhat に接続 | `bridge: listening to 5 merchandise(s)` |
| Step 5 | MetaMask 残高 | `10000 ETH` (Account #2) |
| Step 6-A | bridge が Purchase 検知 | `bridge: claim ok jti=...` |
| Step 6-B | audit に marketplace/claim | `subject=eth:0x3C44...`, `reason=claim_received:...` |
| Step 7-A | wallet で VC 受領 | claims に `merchandise_address` / `tx_hash` / `buyer_eth_addr` |
| Step 7-B | audit に eth_did_bound | `raw_topic=marketplace/issued`, `holder_did=did:jwk:...` |
| Step 8 | ViewerToken 発行 | publisher ログ `viewer_token_issued vc_kind=PurchaseViewerVC` |
| Step 9 | merchandise 逆引き取得 | `{"dataset_id": "home/env/temperature", "read_count": 1}` |
| Step 9 (続) | 60 秒後 401 | `{"detail": "viewer_token_expired"}` |
| Step 10 | audit log 4 行連鎖 | `marketplace/claim` → `marketplace/issued` → `oid4vp/response` → `platform/data` |

---

## トラブルシューティング

実機検証で発生した詰まりポイントとその対処です。

### A. MetaMask が `chainId エラー` で tx を送れない

```
MetaMask - RPC Error: Trying to send a raw transaction with an invalid chainId.
```

**原因**: Hardhat ノードを再起動した後、MetaMask 側に古い nonce / chainId キャッシュが残っている。

**対処**: MetaMask → Settings → Advanced → **「Clear activity tab data」** または **「Reset Account」**。
それでも駄目なら **Step 6 操作 B (Hardhat console)** で購入を代替実行できます。bridge と publisher の動作確認には十分です。

### B. Purchase が `0x295f0a57` で revert

```
Error: VM Exception while processing transaction: reverted with an unrecognized custom error
```

**原因**: `PubKey__NotRegistered`。Step 3 をスキップした、または Account #2 以外で PubKey を登録した。

**対処**: Step 3 を再実行。Hardhat console で `pubKey.registerKey("[...]")` を **buyer (Account #2)** で呼ぶ。

### C. iPhone wallet が deeplink を開いた瞬間に落ちる / 何も表示されない

**原因 1**: bridge から発行された deeplink 内の `credential_issuer` が
`http://publisher:8080` (Docker 内部ホスト名) になっており、iPhone から到達不能。

**対処**: `infra/.env` に `BRIDGE_PUBLIC_PUBLISHER_URL=http://<LAN_IP>:8080` を設定して
bridge を再起動 (Step 4)。deeplink を再取得して、内部にある `credential_issuer` が
LAN IP になっていることを確認:

```bash
DEEPLINK=$(docker logs iw3ip-mv-bridge 2>&1 | grep "claim ok" | tail -1 | sed -E 's/.*deeplink=//')
echo "$DEEPLINK" | python3 -c "import sys,urllib.parse,json; d=urllib.parse.unquote(sys.stdin.read().split('credential_offer=',1)[1]); print(json.loads(d)['credential_issuer'])"
# → http://192.168.68.53:8080  (publisher:8080 ではなく)
```

**原因 2**: wallet が `No script URL provided` エラー画面を出している = Metro bundler 未起動。

**対処**: `cd ~/program/iw3ip-wallet && npx react-native start` を別ターミナルで起動し、
wallet の「Reload JS」を押す。

**原因 3**: wallet 側で issuer metadata が古いキャッシュのまま。

**対処**: iPhone のアプリスイッチャーで wallet を **完全終了** → 再起動。

### D. bridge ログに `Purchase event` が一度も出ない

```
bridge: listening to 5 merchandise(s)
@TODO TypeError: results is not iterable
```

**原因**: 古いコード (filter 購読) が残っている。最新コードでは `getLogs` polling に変更済み。

**対処**:
```bash
cd ~/program/Blockchain_IoT_Marketplace
git checkout main && git pull --ff-only
docker compose -f infra/docker-compose.yml --profile mv-bridge up --build -d bridge
```

### E. iPhone から `192.168.68.53:8080/health` に届かない

**原因**: PC とスマホが別ネットワークか、Mac の LAN IP が変わっている。

**対処**:
```bash
ipconfig getifaddr en0   # 現在の LAN IP
```
変わっていれば `infra/.env` の `BRIDGE_PUBLIC_PUBLISHER_URL` を更新して bridge 再起動。

---

## このハンズオンの限界 (将来課題)

- **eth_addr ↔ did:jwk のなりすまし対策**: MVP は bridge / フロント
  からの POST を信用するだけ。本番は EIP-712 署名で「この tx は
  私のもの」と holder に証明させる必要がある (詳細は[設計仕様 §7.2](../design/marketplace-vc-bridge-spec.md))
- **dataset_id の発見**: 現状フロントで query string にハードコード。
  Merchandise の `additionalInfo` から動的に読む拡張は別タスク
- **ServiceVC との統合**: 連続 MQTT を wallet 化する話 (Stage 4 prep) と、
  購入連動で読む話 (Stage 5) は別レイヤー。両方を組み合わせるシナリオは
  Stage 6 以降

## 関連

- [Marketplace VC Bridge 設計仕様 (v1/v2)](../design/marketplace-vc-bridge-spec.md) — 全体構造
- [SSI Wallet (Stage 1)](ha-ssi-wallet.md) — 書き込み単回認可
- [SSI Viewer (Stage 3)](ha-ssi-viewer.md) — 読み出し多回認可 (v1 dataset 直指定)
- [SSI Service (Stage 4 prep)](ha-ssi-service.md) — M2M 連続書き込み
