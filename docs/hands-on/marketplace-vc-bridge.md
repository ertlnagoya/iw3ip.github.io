# マーケットプレイス × ウォレット連携 (v2 / Stage 5)

!!! abstract "v2 を体験するハンズオン"
    マーケットプレイスでの購入を起点に、買い手のウォレットへ
    PurchaseViewerVC を発行し、提示によってデータを取得する
    end-to-end フローを体験します。
    設計詳細は [Marketplace VC Bridge 設計仕様](../design/marketplace-vc-bridge-spec.md)
    を参照。

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

## つまずきやすい点

- bridge は **host で動く Hardhat** に `host.docker.internal:8545` で繋ぐ
  (compose 内に Hardhat は同居していない)
- iot-market-ui はフロントから直接 `/marketplace/claim` を叩く実装で、
  bridge とフロントの両方が叩いても **冪等性で安全**
- PurchaseViewerVC は ConsentVC / ViewerVC / ServiceVC とは別 VC。
  ウォレット側で 4 種類が別カードとして見える
- 購入後にウォレット未設定だと VC 受領が完了しない (deeplink で wallet が起動する状態が前提)

## 公式リンク

- [Marketplace VC Bridge 設計仕様 (v1/v2)](../design/marketplace-vc-bridge-spec.md)
- [Stage 1: ウォレットサンプル](ha-ssi-wallet.md)
- [Stage 3: ビューワサンプル](ha-ssi-viewer.md)

## 前提

- [Stage 1](ha-ssi-wallet.md) と [Stage 3](ha-ssi-viewer.md) の動作確認が済んでいる
- iot-market-ui (Svelte) と Hardhat ローカルチェーンが起動できる
- iw3ip-wallet が iPhone 実機で動く
- `home/env/temperature` 用の Merchandise が IoTMarket に登録済 (任意の dataset で可)

## 1. 全体像

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

## 2. 起動

### 2-A. Hardhat と Merchandise

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market

# Hardhat ノードを起動 (別ターミナル維持)
npx hardhat node

# (別ターミナル) IoTMarket を deploy + Merchandise を登録
npx hardhat run scripts/deployIoTMarket.ts --network localhost
npx hardhat run scripts/deployMerchandiseWithIoTMarket.ts --network localhost

# IoTMarket のアドレスをメモする (compose 用)
```

### 2-B. publisher + bridge を起動

```bash
cd ~/program/Blockchain_IoT_Marketplace

# IoTMarket のアドレスを env に書く
echo "BRIDGE_IOT_MARKET_ADDRESS=0x..." >> infra/.env

docker compose -f infra/docker-compose.yml \
  --profile ssi-wallet --profile mv-bridge \
  up --build -d

# ヘルスチェック
curl -s http://192.168.68.53:8080/health

# PurchaseViewerVC が登録されているか
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer \
  | python3 -m json.tool | grep -A2 PurchaseViewerVC
```

### 2-C. iot-market-ui を LAN 公開で起動

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market-ui

# publisher の URL を env に
echo "VITE_PUBLISHER_URL=http://192.168.68.53:8080" > .env.local

npm run dev -- --host 0.0.0.0 --port 5173
```

## 3. 購入してみる

### 3-A. PC で

ブラウザで `http://192.168.68.53:5173/` を開き、商品リストから
`home/env/temperature` 系の Merchandise を 1 つ選ぶ。

詳細ページで「Purchase」ボタンを押す。MetaMask が立ち上がるので
ETH 残高がある account で署名・送信。

### 3-B. tx 確定後の自動遷移

tx が confirm されると自動で `/purchased/<txHash>` に遷移します。
ページ上で:

- 「publisher にクレームを登録中…」 → 数秒で「ウォレットで開く」ボタンと
  QR が表示される
- QR の下に `claim_id` / `tx_hash` / `merchandise` / `dataset_id` /
  `buyer_eth_addr` が並ぶ

publisher のログに次のような行が出ているはず:

```
audit: raw_topic=marketplace/claim reason=claim_received:<id>:tx=0x...
```

## 4. iw3ip-wallet で VC を受け取る

スマホ (iPhone) で `/purchased/<txHash>` ページの **QR を読み取る**
か、PC ブラウザで「ウォレットで開く」ボタンを押すと AirDrop で
deeplink を送る。

iw3ip-wallet が deeplink を受け取り「IW3IP Purchase Viewer Credential」
として保存する確認画面が出る。承認すると VC が wallet に保存される。

claim:
- `dataset_id`
- `allowed_actions`: `["read"]`
- `merchandise_address`
- `buyer_eth_addr`
- `tx_hash`

publisher ログで eth↔did 紐付けを確認:

```
audit: raw_topic=marketplace/issued reason=eth_did_bound:claim=<id>:eth=0x...:tx=0x...
holder_did=did:jwk:...
```

これで購入者の Ethereum address と did:jwk が **publisher の audit log
で紐付いた**ことが分かります。

## 5. ViewerToken を取り出す

PC ブラウザで:

```
http://192.168.68.53:8080/verifier/request?dataset_id=home/env/temperature&vc_kind=PurchaseViewerVC
```

QR / AirDrop deeplink → wallet で **PurchaseViewerVC** を選んで提示。

publisher ログに ViewerToken が出力されます:

```bash
docker logs $(docker ps -qf name=publisher) 2>&1 \
  | grep viewer_token_issued | tail -1
# → viewer_token_issued vc_kind=PurchaseViewerVC jti=... token=... ttl=60s
```

## 6. データ取得 (`/platform/data?merchandise=<addr>`)

```bash
TOKEN=<viewer_token>
MERCHANDISE=<merchandise_address>

curl -s -H "Authorization: Bearer $TOKEN" \
  "http://192.168.68.53:8080/platform/data?merchandise=$MERCHANDISE" \
  | python3 -m json.tool
```

期待:
```json
{
  "dataset_id": "home/env/temperature",
  "count": N,
  "read_count": 1,
  "rows": [...]
}
```

dataset_id は **merchandise_address から逆引き**で解決されます。
buyer は dataset 名を覚えていなくても、買った商品の住所だけで取得できます。

## 7. v1 lane との対比

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

## 8. 監査ログの全体像 (1 購入で 4 行)

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=10' | python3 -m json.tool
```

期待される最小 4 行 (新しい順):

| raw_topic | reason | holder_did |
| --- | --- | --- |
| `platform/data` | `viewer_token_used:<jti>:<read_count>` | did:jwk:... |
| `oid4vp/response` | `ok` | did:jwk:... |
| `marketplace/issued` | `eth_did_bound:claim=<id>:eth=...:tx=...` | did:jwk:... |
| `marketplace/claim` | `claim_received:<id>:tx=...` | (空; eth_addr のみ) |

## 9. エラーケース

| 状況 | エラー |
| --- | --- |
| 同じ tx_hash で 2 回 claim | 200, `created: false` (idempotent) |
| 未登録の merchandise で `/platform/data` | 404 `unknown_merchandise:0x...` |
| dataset 不一致のトークンで取得 | 403 `viewer_token_dataset_mismatch` |
| ViewerToken 期限切れ | 401 `viewer_token_expired` (60 秒で切れる) |
| ConsentVC を `/platform/data` に流用 | 401 (ViewerToken namespace に該当無し) |

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
